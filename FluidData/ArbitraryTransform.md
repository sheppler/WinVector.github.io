Arbitrary Data Transforms Using cdata
================
John Mount, Win-Vector LLC
11/22/2017

We have been writing a lot on higher-order data transforms lately:

-   [Coordinatized Data: A Fluid Data Specification](http://winvector.github.io/FluidData/RowsAndColumns.html)
-   [Data Wrangling at Scale](http://winvector.github.io/FluidData/DataWranglingAtScale.html)
-   [Fluid Data](http://winvector.github.io/FluidData/FluidData.html)
-   [Big Data Transforms](http://www.win-vector.com/blog/2017/10/big-data-transforms/).

What I want to do now is "write a bit more, so I finally feel I have been concise."

The [`cdata`](https://winvector.github.io/cdata/) [`R`](https://www.r-project.org) package supplies general data transform operators.

-   The whole system is based on two primitives or operators [`cdata::rowrecs_to_blocks()`](https://winvector.github.io/cdata/reference/rowrecs_to_blocks.html) and [`cdata::blocks_to_rowrecs()`](https://winvector.github.io/cdata/reference/blocks_to_rowrecs.html).
-   These operators have pivot, un-pivot, one-hot encode, transpose, moving multiple rows and columns, and many other transforms as simple special cases.
-   It is easy to write many different operations in terms of the `cdata` primitives.
-   These operators can work-in memory or at big data scale (with databases and Apache Spark; for big data we use the [`cdata::rowrecs_to_blocks()`](https://winvector.github.io/cdata/reference/rowrecs_to_blocks.html) and [`cdata::blocks_to_rowrecs_q()`](https://winvector.github.io/cdata/reference/blocks_to_rowrecs_q.html) variants).
-   The transforms are controlled by a control table that itself is a diagram of or picture of the transform.

We will end with a quick example, centered on pivoting/un-pivoting values to/from more than one column at the same time.

Suppose we had some sales data supplied as the following table:

| SalesPerson | Period |  BookingsWest|  BookingsEast|
|:------------|:-------|-------------:|-------------:|
| a           | 2017Q1 |           100|           175|
| a           | 2017Q2 |           110|           180|
| b           | 2017Q1 |           250|             0|
| b           | 2017Q2 |           245|             0|

Suppose we are interested in adding a derived column: which region the salesperson made most of their bookings in.

``` r
library("cdata")
```

    ## Loading required package: wrapr

``` r
library("seplyr")
```

``` r
d <- d  %.>% 
  dplyr::mutate(., BestRegion = ifelse(BookingsWest > BookingsEast, 
                                       "West",
                                       ifelse(BookingsEast > BookingsWest, 
                                              "East", 
                                              "Both")))
```

Our notional goal is (as part of a larger data processing plan) to reformat the data a thin/tall table or a [RDF-triple](https://en.wikipedia.org/wiki/Semantic_triple) like form. Further suppose we wanted to copy the derived column into every row of the transformed table (perhaps to make some other step involving this value easy).

We can use [`cdata::rowrecs_to_blocks()`](https://winvector.github.io/cdata/reference/rowrecs_to_blocks.html) to do this quickly and easily.

First we design what is called a transform control table.

``` r
cT1 <- data.frame(Region = c("West", "East"),
                  Bookings = c("BookingsWest", "BookingsEast"),
                  BestRegion = c("BestRegion", "BestRegion"),
                  stringsAsFactors = FALSE)
print(cT1)
```

    ##   Region     Bookings BestRegion
    ## 1   West BookingsWest BestRegion
    ## 2   East BookingsEast BestRegion

In a control table:

-   The column names specify new columns that will be formed by `cdata::rowrecs_to_blocks()`.
-   The values specify where to take values from.

This control table is called "non trivial" as it does not correspond to a simple pivot/un-pivot (those tables all have two columns). The control table is a picture of of the mapping we want to perform.

An interesting fact is `cdata::blocks_to_rowrecs(cT1, cT1, keyColumns = NULL)` is a picture of the control table as a one-row table (and this one row table can be mapped back to the original control table by `cdata::rowrecs_to_blocks()`, these two operators work roughly as inverses of each other; though `cdata::rowrecs_to_blocks()` operates on rows and [`cdata::blocks_to_rowrecs()`](https://winvector.github.io/cdata/reference/blocks_to_rowrecs.html) operates on groups of rows specified by the keying columns).

The mnemonic is:

-   `cdata::blocks_to_rowrecs()` converts arbitrary grouped blocks of rows that look like the control table into many columns.
-   `cdata::rowrecs_to_blocks()` converts each row into row blocks that have the same shape as the control table.

Because pivot and un-pivot are fairly common needs `cdata` also supplies functions that pre-populate the controls tables for these operations ([`buildPivotControlTableD()`](https://winvector.github.io/cdata/reference/buildPivotControlTableD.html) and [`buildUnPivotControlTable()`](https://winvector.github.io/cdata/reference/buildUnPivotControlTable.html)).

To design any transform you draw out the control table and then apply one of these operators (you can pretty much move from any block structure to any block structure by chaining two or more of these steps).

We can now use the control table to supply the same transform for each row.

``` r
d  %.>% 
  dplyr::mutate(., 
                Quarter = substr(Period,5,6),
                Year = as.numeric(substr(Period,1,4)))  %.>% 
  dplyr::select(., -Period)  %.>% 
  rowrecs_to_blocks(., 
                    controlTable = cT1, 
                    columnsToCopy = c('SalesPerson', 
                                      'Year', 
                                      'Quarter')) %.>% 
  arrange_se(., c('SalesPerson', 'Year', 
                  'Quarter', 'Region'))  %.>% 
  knitr::kable(.)  
```

| SalesPerson |  Year| Quarter | Region |  Bookings| BestRegion |
|:------------|-----:|:--------|:-------|---------:|:-----------|
| a           |  2017| Q1      | East   |       175| East       |
| a           |  2017| Q1      | West   |       100| East       |
| a           |  2017| Q2      | East   |       180| East       |
| a           |  2017| Q2      | West   |       110| East       |
| b           |  2017| Q1      | East   |         0| West       |
| b           |  2017| Q1      | West   |       250| West       |
| b           |  2017| Q2      | East   |         0| West       |
| b           |  2017| Q2      | West   |       245| West       |

Notice we were able to easily copy the extra `BestRegion` values into all the correct rows.

It can be hard to figure out how to specify such a transformation in terms of pivots and un-pivots. However, as we have said: by drawing control tables one can easily design and manage fairly arbitrary data transform sequences (often stepping through either a denormalized intermediate where all values per-instance are in a single row, or a thin intermediate like the triple-like structure we just moved into).
