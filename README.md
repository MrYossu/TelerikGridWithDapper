# Using Dapper with the Telerik Blazor grid

>**Note:** I added the extension method described here into version 1.9.0 of the [Pixata.Blazor.TelerikComponents](https://github.com/MrYossu/Pixata.Utilities/tree/master/Pixata.Blazor.TelerikComponents) Nuget package. However, it will probably be removed from the next version, as I later discovered that I can do everything that this method does without needing Dapper. I have written a new extension method that works with EF Core, and has more functionality. You can see [the sample project for that here](https://github.com/MrYossu/TelerikGridWithFromSql), and a blog post should follow soon with more explanation.

We were having problems with the slow loading time for data in a grid. Even though we had created a flat table for the data and added indexes, etc, there was still a noticeable delay in loading the data.

I decided to try using [Dapper](https://github.com/DapperLib/Dapper) to improve the performance. One of the main selling points of Dapper is its speed. Sounds good to me!

The problem was that it wasn't obvious how to use Dapper with the Telerik grid, and no-one at Telerik had any experience with it. In principle it was straightforward, but in practice, less so. Judging by the documentation Telerik have on loading data manually, we needed to handle the grid's `OnRead` handler, but it wasn't at all clear what we neded to do there.

After much gnashing of teeth and pulling out of what little hair I have left, I managed to come up with an extension method to the `GridReadEventArgs` object that is passed to the `OnRead` handler. I won't bore you with the details, as the method is (hopefully) easy enough to use without needing to know how it works, and is also (hopefully) well commented enough for you to understand what's going on, in case you want to modify it.

This repository contains a helper class with the extension method, as well as a sample component showing how to use it. The sample assumes you have the ubiquitous Northwind database loaded on your local server. If not, you'll either need to [download the database](https://github.com/Microsoft/sql-server-samples/tree/master/samples/databases/northwind-pubs) or modify the code to point at a different database.

## How to use the extension method

Here is the markup for the grid...

```xml
<TelerikGrid TItem="@Product"
             OnRead="@LoadData"
             ScrollMode="@GridScrollMode.Virtual"
             FilterMode="@GridFilterMode.FilterRow"
             Height="500px"
             RowHeight="40"
             Sortable="true"
             PageSize="15"
             FilterRowDebounceDelay="350">
  <GridColumns>
    <GridColumn Field="@nameof(Product.ProductName)" />
    <GridColumn Field="@nameof(Product.UnitPrice)" />
    <GridColumn Field="@nameof(Product.UnitsInStock)" />
  </GridColumns>
</TelerikGrid>
```

Nothing exciting there. The only line that needs any comment is the handler for `OnRead`. All that's actually needed to get this to work is a one-liner...

```c#
private async Task LoadData2(GridReadEventArgs args) =>
  await args.GetData<Product>(connStr, "Products", "ProductName");
```

The method is generic to the type that you use in the grid. The arguments to the method are...

1. The connection string, in this case defined in the wittily-named `connStr` variable
2. The name of the table to be queried.
3. The name of the default column to be used for sorting.
4. You can add an optional argument to specify the sort direction, which by default is ascending. This argument is a value from the Telerik `ListSortDirection` `enum`

There is a further optional parameter, but I'll leave that for the moment to keep things clear.

That's all you need. The grid will load the data, and will respect the filtering and sorting options you set.

As a comparison, our original grid code took about 12-14 seconds to load the data (ulp). By contrast, the version that used this extension method took less than 2 seconds 😎.

## The return values
The code above ignores the values returned from the `GetData` method. Most of the time that's fine, but you may want to do something extra with the data.

As a (slightly spurious) example, suppose you wanted to show some aggregate data below the grid. We will show the number of matching products, and the total value of their stock (ie the unit cost multiplied by the cost per unit).

When using EF Core, this is fairly easy with the [built-in aggregates feature](https://docs.telerik.com/blazor-ui/components/grid/grouping/aggregates). When using Dapper, you have to rool your own code. The `GetData` method returns the information you need to do this.

The return value is a 3-tuple that contains...
1. The number of rows that match the current filtering (which will be the total number of rows in the table if the user didn't set any filtering)
2. The SQL needed to query the database with the same filtering
3. The data values needed to pass as parameters

So, to add our aggregates, we define two properties to hold the values...

```c#
private int MatchingRows { get; set; }
private decimal TotalValue { get; set; }
```

We then capture the return values from the `GetData` method. As the method returns the number of rows, we can assign that directly to `MatchingRows`. We use the other two values to get the total value of stock...

```c#
private async Task LoadData(GridReadEventArgs args) {
  (MatchingRows, string sqlFilters, Dictionary<string, object> values) =
    await args.GetData<Product>(connectionString, "Products", "ProductName");
  await using SqlConnection connection = new(connectionString);
  string sql = $"select sum(unitsinstock * unitprice) from products{sqlFilters}";
  TotalValue = await connection.QuerySingleAsync<decimal>(sql, values);
}
```

## Custom filtering
Sometimes you want to build your own filter for the Telerik grid. For example, we like using the filter row feature, but this only allows you to filter on one value, eg data whose date is after the entered value. What it doesn't allow you to do is filter on a range, eg data whose date is between two dates.

In such cases, you can build your own filter using the `<FilterCellTemplate>` tag, [as shown in the demos](https://demos.telerik.com/blazor-ui/grid/custom-filter-row).

Whilst this works fine when using EF Core, it doesn't work with Dapper, as all the data access is done manually, and the grid doesn't know how to query the database. In this case, you need to tell the grid to rebind the data, passing your extra data values to the `GetData` method.

I haven't included the full code for the customer filter here (due to lack of time), but all that is needed is to set up a collection of `TelerikGridFilterOptions` objects, and pass them to `GetData`. Assuming your custom filter calls the grid's `Rebind` method, then the `OnRead` handler can include those in the call to `GetData`.

First set up a collection of `TelerikGridFilterOptions` objects...

```c#
private List<TelerikGridFilterOptions> _extraFilterOptions = new();
```

Then, in your custom filter code, populate this collection...

```c#
private void FilterDateColumn() {
  _extraFilterOptions = new();
  if (FromSelectedDateTime.HasValue) {
    _extraFilterOptions.Add(new("Date", FromSelectedDateTime.Value, FilterOperator.IsGreaterThanOrEqualTo));
  }
  if (ToSelectedDateTime.HasValue) {
    _extraFilterOptions.Add(new("Date", ToSelectedDateTime.Value, FilterOperator.IsLessThanOrEqualTo));
  }
  _grid.Rebind();
}
```

This example assumes you have two date pickers, one for the "from" date and one for the "to" date.

Then, the call to `GetData` can include the collection, and all just works...

```c#
await args.GetData<TransactionOverview>(connectionString, "TransactionOverviews", "Date", _extraFilterOptions);
```

## Known limitations
As mentioned at the top, this is a work in progress. As time goes on, I may modify it, hopefully adding extra functionality without making it to complex.

In the meantime, I know of the following issues...

- The code here was written with SQL Server in mind. I haven't checked it against any other database. Whilst most of the SQL it produces should be fairly vanilla, I think the SQL used for paging may be specific to SQL Server. If you want to use this code with other databases, you'd need to check if `offset (@Skip) rows fetch next (@PageSize) rows only` will work. If not, you'll need to modify the SQL
- Optimisation freaks may note that the SQL generated starts with `select * from`, and get upset that this is inefficient. Whilst using the wildcard is marginally slower than specifying the columns to be selected, my testing showed it to be so close that it wasn't worth complicating the method with an extra parameter to specify the columns. If this bothers you, or you hit a situation where this micro-optimisation does make a difference, feel free to copy the code and add the parameter.