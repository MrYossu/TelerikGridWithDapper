# Using Dapper with the Telerik Blazor grid

We were having problems with the slow loading time for data in a grid. Even though we had created a flat table for the data and added indexes, etc, there was still a noticeable delay in loading the data.

I decided to try using [Dapper](https://github.com/DapperLib/Dapper) to improve the performance. One of the main selling points of Dapper is its speed. Sounds good to me!

The problem was that it wasn't obvious how to use Dapper with the Telerik grid, and no-one at Telerik had any experience with it. In principle it was straightforward, but in practice, less so. Judging by the documentation Telerik have on loading data manually, we needed to handle the grid's `OnRead` handler, but it wasn't at all clear what we neded to do there.

After much gnashing of teeth and pulling out of what little hair I have left, I managed to come up with an extension method to the `GridReadEventArgs` object that is passed to the `OnRead` handler. I won't bore you with the details, as the method is (hopefully) easy enough to use without needing to know how it works, and is also (hopefully) well commented enough for you to understand what's going on, in case you want to modify it.

This repository contains
