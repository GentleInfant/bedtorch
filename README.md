
<!-- README.md is generated from README.Rmd. Please edit that file -->

# bedtorch

<!-- badges: start -->

[![R-CMD-check](https://github.com/haizi-zh/bedtorch/workflows/R-CMD-check/badge.svg)](https://github.com/haizi-zh/bedtorch/actions)
[![test-coverage](https://github.com/haizi-zh/bedtorch/workflows/test-coverage/badge.svg)](https://github.com/haizi-zh/bedtorch/actions)
<!-- badges: end -->

## Motivation

The goal of bedtorch is to provide a fast and native toolsuite for BED
file manipulation.

In bioinformatics studies, an important type of jobs is related to BED
and BED-like file manipulation. To name a few example:

-   Filtering cfDNA fragments based on length
-   Identifying genomic regions overlapping with certain genes
-   Filtering features based on epigenetic profiles such as histone
    modification levels
-   Extracting SNVs in a certain genomic interval

Many of such tasks can be done by some highly-optimized tools, such as
[bedtools](https://bedtools.readthedocs.io/en/latest/) and
[bedops](https://bedops.readthedocs.io/en/latest/). However, these are
binary tools which only works in shell. Sometimes, especially for
developers using R, it’s desirable to invoke these tools in R codes.

One solution is using functions that invokes system commands
([`system2`](https://www.rdocumentation.org/packages/base/versions/3.6.2/topics/system2)
or
[`system`](https://www.rdocumentation.org/packages/base/versions/3.6.2/topics/system)).
[`bedr`](https://cran.r-project.org/web/packages/bedr/vignettes/Using-bedr.html),
a popular package, is one good example. It’s a bridge linking between R
codes and shell tools. Under the hood, it writes data into the temporary
directory, calls certain command line tools such as `tabix`, `bedtools`,
or `bedops`, parses the output and re-generate a data frame in R space.

Generally this approach is very helpful. However, it’s worth noting some
disadvantages.

-   This workflow requires lots of disk IO, since data frames in R needs
    to be written to temporary directories before being fed to command
    line tools.
-   Command line output is captured and transferred back to R s space.
    The data transfer is done by stdin/stdout pipes. For R, reading
    large amount of text content from stdin can be extremely slow. As a
    result, most of time when large dataset is involved, `bedr` is
    hardly usable.
-   `bedr` captures command line output as plain text. Therefore, when
    the data is transferred back to R, most of the information about
    data types, index (in the case of `data.table`), column names, etc.,
    are lost. It’s the package user’s responsibility to make sure data
    types are correct, column names are restored, column indices have
    been rebuilt (in the case of `data.table`, which can be very time
    consuming). Often this job is tedious, error-prone and introduces
    very large overheads.
-   It requires the command line tools to be present in `PATH`. This
    depends on the end user’s system environment and the R developer can
    do nothing about it. It puts additional burden to deployment,
    especially when the end user is not an IT professional.

Another approach is using native R data types to represent BED-like
data. R’s native `data.frame` seems to be a good model, but in many
scenarios, it’s performance is poor. More importantly, it lacks many
advanced features compared with `dplyr` and `data.table`.

It would be desirable to have an R package, which store BED-like data as
`data.table` objects, and provide native support for common BED
manipulations, such as merge, intersect, filter, etc.

## Details

`bedtorch` is an attempt for this. Under the hood, every BED dataset is
represented as a `data.table` object. Users can apply various operations
on it, such as intersect, merge, etc. Users can also perform any
`data.table` operations as needed.

In terms of the core computation, `bedtorch`’s performance is comparable
to `bedtools`. However, since no disk IO and data conversion is needed,
users can directly manipulate the dataset in memory, therefore in
practice, via `bedtorch`, many tasks can be done about one magnitude
faster.

Additionally, `bedtorch` can write dataset directly to BGZIP-format
files. [BGZIP](http://www.htslib.org/doc/bgzip.html) is a variant of
gzip and is widely used in bioinformatics. It’s compatible with gzip,
with an important additional feature: allows a compressed file to be
indexed for fast query. However, R is unaware of BGZIP. It considers all
`.gz` files as gzip-compressed, thus doesn’t take advantage of BED
index, and cannot output BGZIP files.

To address this, `bedtorch` can directly write a dataset to disk in
BGZIP format, and (optionally) create the index.

### Installation

``` r
# install.packages("devtools")
devtools::install_github("haizi-zh/bedtorch")
```

### Example

This is a basic example which shows you how to solve a common problem:

``` r
library(bedtorch)

## Load BED data
bedtbl <-
  read_bed(system.file("extdata", "example2.bed.gz", package = "bedtorch"),
           range = "1:3001-4000")
head(bedtbl)
#>    chrom start  end score1 score2
#> 1:     1  2925 3011    106    181
#> 2:     1  3003 3092     88    193
#> 3:     1  3091 3193    118    212
#> 4:     1  3164 3248     94    211
#> 5:     1  3232 3345    107    205
#> 6:     1  3300 3395     88    193

## Merge intervals, and take the mean of score in each merged group
merged <- merge_bed(bedtbl,
                    operation = list(
                      mean_score = function(x)
                        mean(x$score1)
                    ))
head(merged)
#>    chrom start  end mean_score
#> 1:     1  2925 4016   99.07143

## Find intersections between two datasets
tbl_x <- read_bed(system.file("extdata", "example_merge.bed", package = "bedtorch"))
tbl_y <- read_bed(system.file("extdata", "example_intersect_y.bed", package = "bedtorch"))
head(intersect_bed(tbl_x, tbl_y))
#>    chrom start end score
#> 1:    21    22  25     7
#> 2:    21    26  30     7
#> 3:    21    29  35     9
#> 4:    21    47  49     1
#> 5:    21    47  50     2
#> 6:    21    53  55     5
```

### Performance

The following is the result of a simple benchmark for three common BED
manipulations: merge, intersect and map.

Benchmark platform information:

-   OS: macOS Big Sur 11.1
-   CPU: 2.7 GHz Quad-Core Intel Core i7
-   Memory: 16 GB LPDDR3

![](https://raw.githubusercontent.com/haizi-zh/bedtorch/main/data-raw/benchmark.png "Benchmark")

From the benchmark result, we can see that for all three tasks,
`bedtorch` uses much less time than `bedtools`.

Note: this does not mean `bedtools` is indeed slower than `bedtorch` at
performing the actual computation. Don’t forget `bedtools` requires
large amount of disk IO, while `bedtorch` does not.

## Reference

For more details, refer to the documentation page:
<https://haizi-zh.github.io/bedtorch/>

Many features are inspired by bedtools. Thus, it’s helpful to get
familiar with bedtool’s documentation:
<https://bedtools.readthedocs.io/en/latest/index.html>
