+++
title = "Modify and aggregate dataframes in Chez Scheme"
date = 2020-09-05
updated = 2021-04-26
[taxonomies]
categories = ["Chez Scheme", "dataframe"]
tags = ["dataframe", "data-structures", "association-list", "modify", "modify-at", "dplyr", "mutate", "mutate_at", "aggregate", "macros"]
+++

This post is part of a [series](/categories/dataframe/) on the [dataframe library](https://github.com/hinkelman/dataframe/) for Chez Scheme. In this post, I will contrast the `dataframe` library with functions from base R and the [`dplyr` package](https://dplyr.tidyverse.org) for modifying and aggregating dataframes.

<!-- more -->

### Set up

First, let's create a dataframe in both languages.

```
df <- data.frame(grp = c("a", "a", "b", "b", "b"),
                 trt = c("x", "y", "x", "y", "y"),
                 adult = c(1, 2, 3, 4, 5),
                 juv = c(10, 20, 30, 40, 50))
                 
(define df
  (make-dataframe '((grp "a" "a" "b" "b" "b")
                    (trt "x" "y" "x" "y" "y")
                    (adult 1 2 3 4 5)
                    (juv 10 20 30 40 50))))
```

### Mutate/Modify [[1]](#1)

In R, `dplyr::mutate` changes all the values in a column according to the expression provided. If the column name exists in the dataframe, then the old column is overwritten. If the column name doesn't exist, then a new column is created at the end of the dataframe. A scalar value is filled across all rows.

```
> df2 <- dplyr::mutate(df, 
                       grp = toupper(grp),
                       total = adult + juv,
                       scalar = 16,
                       lst = c(2, 4, 6, 8, 10))

> df2
  grp trt adult juv total scalar lst
1   A   x     1  10    11     16   2
2   A   y     2  20    22     16   4
3   B   x     3  30    33     16   6
4   B   y     4  40    44     16   8
5   B   y     5  50    55     16  10
```

`dataframe-modify` takes a `modify-expr` (see more below) to replicate the core behavior of `dplyr::mutate`. When passing values directly (e.g., scalar or list with length equal to number of rows), the column names used in the expression need to be explicitly specified as missing with `()`.

```
> (define df2
    (dataframe-modify df (modify-expr (grp (grp) (string-upcase grp))
                                      (total (adult juv) (+ adult juv))
                                      (scalar () 16)
                                      (lst () '(2 4 6 8 10)))))
> (dataframe-display df2)
 dim: 5 rows x 7 cols
   grp   trt  adult   juv  total  scalar   lst 
     A     x     1.   10.    11.     16.    2. 
     A     y     2.   20.    22.     16.    4. 
     B     x     3.   30.    33.     16.    6. 
     B     y     4.   40.    44.     16.    8. 
     B     y     5.   50.    55.     16.   10. 
```

`dplyr` also provides `mutate_at` and `mutate_all` [[2]](#2).

```
> dplyr::mutate_at(df2, c("total", "scalar", "lst"), sqrt)
  grp trt adult juv    total scalar      lst
1   A   x     1  10 3.316625      4 1.414214
2   A   y     2  20 4.690416      4 2.000000
3   B   x     3  30 5.744563      4 2.449490
4   B   y     4  40 6.633250      4 2.828427
5   B   y     5  50 7.416198      4 3.162278
```

`dataframe-modify-at` works similarly, but the procedure, `sqrt` in this example, is constrained to only accept one argument.

```
> (dataframe-display
   (dataframe-modify-at df2 sqrt 'total 'scalar 'lst))
 dim: 5 rows x 7 cols
   grp   trt  adult   juv   total  scalar     lst 
     A     x     1.   10.  3.3166      4.  1.4142 
     A     y     2.   20.  4.6904      4.  2.0000 
     B     x     3.   30.  5.7446      4.  2.4495 
     B     y     4.   40.  6.6332      4.  2.8284 
     B     y     5.   50.  7.4162      4.  3.1623 
```

#### Implementation

`modify-expr` is a macro that allows for a more concise syntax when writing the expressions used to modify a dataframe. 

```
(define-syntax modify-expr
  (syntax-rules ()
    [(_ (new-name names expr) ...)
     (list
      (list (quote new-name) ...)
      (list (quote names) ...)
      (list (lambda names expr) ...))]))
```

The following `modify-expr` 

```
(modify-expr (grp (grp) (string-upcase grp))
             (total (adult juv) (+ adult juv))
             (scalar () 16)
             (lst () '(2 4 6 8 10)))
```

expands to

```
'((grp total scalar lst)
  ((grp) (adult juv) () ())
  (lambda (grp) (string-upcase grp))
  (lambda (adult juv) (+ adult juv))
  (lambda () 16)
  (lambda () '(2 4 6 8 10)))
```

In [previous posts](/categories/dataframe/) on macros used in filtering and sorting dataframes, I've acknowledged that the `filter-expr` and `sort-expr` macros don't provide a very compelling simplification. In this case, though, the `modify-expr` macro is helpful both for reducing the number of characters and keeping the pieces of the expression together. The use of `filter-expr` and `sort-expr` follows from `modify-expr` and provides more consistent syntax across all of the dataframe procedures.

### Aggregate

In base R, dataframes are aggregated by first splitting into groups, applying the summary statistic, and then complining the pieces. This example uses the formula syntax where the left side indicates the columns to be summarized and the right side indicates the grouping variables.

```
> aggregate(cbind(adult, juv) ~ grp + trt, data = df, sum)
  grp trt adult juv
1   a   x     1  10
2   b   x     3  30
3   a   y     2  20
4   b   y     9  90
```

`dataframe-aggregate` also uses the split-apply-combine approach, but uses similar syntax to `dataframe-modify`. 

```
> (dataframe-display
   (dataframe-aggregate df
                        '(grp trt)
                         (aggregate-expr (adult-sum (adult) (apply + adult))
                         (juv-sum (juv) (apply + juv)))))
 dim: 4 rows x 4 cols
   grp   trt  adult-sum  juv-sum 
     a     x         1.      10. 
     a     y         2.      20. 
     b     x         3.      30. 
     b     y         9.      90. 
```

***

<a name="1"></a> [1] I opted not to use the term mutate in Chez's `datafame` library because it felt too directly contradictory with the fact that dataframes are immutable. Arguably, this was a silly choice because modify is a synonym of mutate.

<a name="2"></a> [2] These functions have been superseded by the use of `across` within `mutate`.
