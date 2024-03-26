+++
title =  "Split, bind, and append dataframes in Scheme"
date = 2020-04-04
updated = 2024-03-12
[taxonomies]
categories = ["dataframe", "Scheme", "Chez Scheme"]
tags = ["dataframe", "data-structures", "association-list", "replicate", "rep", "cbind", "dplyr", "bind_rows"]
+++

This post is the third in a [series](/categories/dataframe/) on the [`dataframe` library](https://github.com/hinkelman/dataframe/) for Scheme (R6RS). In this post, I will contrast the `dataframe` library with functions from base R and the [`dplyr` package](https://dplyr.tidyverse.org) for splitting, binding, and appending dataframes.

<!-- more -->

### Set up

First, let's create a couple of dataframes in both languages.

```
df1 <- data.frame(trt = rep(c("a", "b"), each = 6),
                  grp = rep(rep(c("x", "y"), each = 3), times = 2),
                  rsp = rep(1:4, each = 3))

df2 <- data.frame(asc = 0:11, desc = 11:0)
                 
(define df1
  (make-dataframe
   (list (make-series 'trt (rep '("a" "b") 6 'each))
         (make-series 'grp (rep (rep '("x" "y") 3 'each) 2 'times))
         (make-series 'rsp (rep '(1 2 3 4) 3 'each)))))

(define df2
  (make-dataframe
   (list (make-series 'asc (iota 12))
         (make-series 'desc (reverse (iota 12))))))
```

### Append

I'm using `append` to refer to a `cbind` operation in R.

```
> head(cbind(df1, df2))

  trt grp rsp asc desc
1   a   x   1   0   11
2   a   x   1   1   10
3   a   x   1   2    9
4   a   y   2   3    8
5   a   y   2   4    7
6   a   y   2   5    6
```

In `dataframe`, we append dataframes with equal numbers of rows via `dataframe-append`.

```
> (dataframe-display (dataframe-append df1 df2) 6)

 dim: 12 rows x 5 cols
   trt   grp   rsp   asc  desc 
     a     x    1.    0.   11. 
     a     x    1.    1.   10. 
     a     x    1.    2.    9. 
     a     y    2.    3.    8. 
     a     y    2.    4.    7. 
     a     y    2.    5.    6. 
```

I chose `dataframe-append` as the name because lists are straightforwardly combined with `append` in Scheme.

```
> (append '(1 2 3) '(4 5 6))

(1 2 3 4 5 6)
```

### Split

In R, `split` returns a named list of dataframes where the names are based on the column names defining the groups.

```
> split(df1, list(df1$trt, df1$grp))

$a.x
  trt grp rsp
1   a   x   1
2   a   x   1
3   a   x   1

$b.x
  trt grp rsp
7   b   x   3
8   b   x   3
9   b   x   3

$a.y
  trt grp rsp
4   a   y   2
5   a   y   2
6   a   y   2

$b.y
   trt grp rsp
10   b   y   4
11   b   y   4
12   b   y   4
```

`dataframe-split` returns a list of dataframes.

```
> (for-each dataframe-display (dataframe-split df1 'trt 'grp))

 dim: 3 rows x 3 cols
     trt     grp     rsp 
   <str>   <str>   <num> 
       a       x      1. 
       a       x      1. 
       a       x      1. 
 dim: 3 rows x 3 cols
     trt     grp     rsp 
   <str>   <str>   <num> 
       a       y      2. 
       a       y      2. 
       a       y      2. 
 dim: 3 rows x 3 cols
     trt     grp     rsp 
   <str>   <str>   <num> 
       b       x      3. 
       b       x      3. 
       b       x      3. 
 dim: 3 rows x 3 cols
     trt     grp     rsp 
   <str>   <str>   <num> 
       b       y      4. 
       b       y      4. 
       b       y      4. 
```

### Bind

For binding by rows, we will use functions from `dplyr`. In the first example, all dataframes in the list have the same columns. 

```
> dplyr::bind_rows(split(df1, list(df1$trt, df1$grp)))

   trt grp rsp
1    a   x   1
2    a   x   1
3    a   x   1
4    b   x   3
5    b   x   3
6    b   x   3
7    a   y   2
8    a   y   2
9    a   y   2
10   b   y   4
11   b   y   4
12   b   y   4
```

`dataframe-bind-all` works similarly to `dplyr::bind_rows`.

```
> (dataframe-display (dataframe-bind-all (dataframe-split df1 'trt 'grp)) 12)

 dim: 12 rows x 3 cols
     trt     grp     rsp 
   <str>   <str>   <num> 
       a       x      1. 
       a       x      1. 
       a       x      1. 
       a       y      2. 
       a       y      2. 
       a       y      2. 
       b       x      3. 
       b       x      3. 
       b       x      3. 
       b       y      4. 
       b       y      4. 
       b       y      4. 
```

To show how to bind dataframes with different columns, let's split up `df1`.

```
df_a <- dplyr::filter(df1, trt == "a")
df_b <- dplyr::filter(df1, trt == "b")

(define-values (df-a df-b)
  (dataframe-partition* df1 (trt) (string=? trt "a")))
```

`dplyr::bind_rows` fills missing columns with `NA`. 

```
dplyr::bind_rows(df_a, df_b[,c("trt", "grp")])

   trt grp rsp
1    a   x   1
2    a   x   1
3    a   x   1
4    a   y   2
5    a   y   2
6    a   y   2
7    b   x  NA
8    b   x  NA
9    b   x  NA
10   b   y  NA
11   b   y  NA
12   b   y  NA
```

Because Scheme doesn't have explicit missing values, `dataframe` uses `'na` to indicate missing values. In `dataframe-bind` (and `dataframe-bind-all`), missing values are filled with `'na` by default, but a different fill value can be specified.

```
> (dataframe-display (dataframe-bind df-a (dataframe-drop* df-b rsp)) 12)

 dim: 12 rows x 3 cols
     trt     grp     rsp 
   <str>   <str>   <num> 
       a       x       1 
       a       x       1 
       a       x       1 
       a       y       2 
       a       y       2 
       a       y       2 
       b       x      na 
       b       x      na 
       b       x      na 
       b       y      na 
       b       y      na 
       b       y      na 
```

### Final thoughts

With the exception of `dataframe-split`, all of the procedures described in the first three posts in the [dataframe series](/categories/dataframe/) involve straightforward composition of Scheme's fundamental procedures (e.g., `map`, `apply`, `append`, `cons`, `car`, `cdr`, etc.) on Scheme's core data structure, i.e., lists. The next couple of posts involve procedures that forced me to wrestle with tradeoffs between convenient syntax via macros (e.g., `dataframe-partition*`) and familiarity/consistency with Scheme's standard library. In the next [post](/filter-partition-and-sort-dataframes-in-scheme/), I will describe how to filter, partition, and sort dataframes in Scheme. 
