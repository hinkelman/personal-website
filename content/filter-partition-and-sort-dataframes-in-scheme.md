+++
title = "Filter, partition, and sort dataframes in Scheme"
date = 2020-04-09
updated = 2024-03-11
[taxonomies]
categories = ["dataframe", "Scheme", "Chez Scheme"]
tags = ["dataframe", "data-structures", "association-list", "dplyr", "arrange", "macros"]
+++

This post is the fourth in a [series](/categories/dataframe/) on the [dataframe library](https://github.com/hinkelman/dataframe/) for Scheme (R6RS). In this post, I will contrast the `dataframe` library with functions from the [`dplyr` R package](https://dplyr.tidyverse.org) for filtering, partitioning, and sorting dataframes. And discuss implementation decisions in the `dataframe` library.

<!-- more -->

### Set up

First, let's create a dataframe in both languages.

```
df <- data.frame(trt = rep(c("a", "b"), each = 6),
                 grp = rep(rep(c("x", "y"), each = 3), times = 2),
                 rsp = rep(1:4, each = 3),
                 ind = 0:11)
                 
(define df
  (make-dataframe
   (list (make-series 'trt (rep '("a" "b") 6 'each))
         (make-series 'grp (rep (rep '("x" "y") 3 'each) 2 'times))
         (make-series 'rsp (rep '(1 2 3 4) 3 'each))
         (make-series 'ind (iota 12)))))
```

### Filter

In R, `dplyr::filter` takes an expression and returns the rows of the dataframe where the expression is `TRUE`.

```
> dplyr::filter(df, trt == "a" & grp == "y")
  trt grp rsp ind
1   a   y   2   3
2   a   y   2   4
3   a   y   2   5
```

Similarly, `dataframe-filter*` returns the rows where the `expr` is `#t`. `dataframe-filter*` is a macro (as are all procedures in `dataframe` with an `*` in the name) that calls the more verbose `dataframe-filter` procedure.

```
> (dataframe-display
   (dataframe-filter*
    df
    (trt grp)
    (and (string=? trt "a")
         (string=? grp "y"))))

 dim: 3 rows x 4 cols
     trt     grp     rsp     ind 
   <str>   <str>   <num>   <num> 
       a       y      2.      3. 
       a       y      2.      4. 
       a       y      2.      5. 
```

#### Implementation

`dataframe-filter*` is a macro (as are all procedures in `dataframe` with an `*` in the name) that allows for a slightly more concise syntax when writing the expressions used to filter a dataframe. `df-filter` is the same internal function called by the more verbose `dataframe-filter`.

```
(define-syntax dataframe-filter*
  (syntax-rules ()
    [(_ df names expr)
     (df-filter df (quote names) (lambda names expr)
     "(dataframe-filter* df names expr)")]))
```

I spent a lot of time wrestling with whether I should use `eval` or macros to simplify the syntax in my `dataframe` procedures. Or whether I should just stick to passing lambda expressions around. I pretty quickly concluded that I should avoid `eval` thanks to some guidance from [Reddit](https://www.reddit.com/r/scheme/comments/e0lj08/lambda_eval_and_macros/) and was intrigued by some suggested neat tricks that didn't involve `eval` or macros. Eventually, though, a better phrased [StackOverflow question](https://stackoverflow.com/questions/60625913/chez-scheme-macro-for-hiding-lambdas) prompted comments and answers that gave me clarity on understanding simple macros.

The following are equivalent

```
(dataframe-filter*
 df
 (trt grp)
 (and (string=? trt "a")
      (string=? grp "y")))

(dataframe-filter
 df
 '(trt grp)
 (lambda (trt grp) (and (string=? trt "a")
                        (string=? grp "y"))))
```

Admittedly, that is not a very compelling simplification. My primary concern with the expanded form was that passing names (e.g., `(trt grp)`) separately and as part of the `lambda` expression introduces a potential source of errors (e.g, names provided in different order). 

The `procedure` is mapped over the dataframe columns identified by `names` and returns a list of boolean values. The boolean values are used to filter each column. Because `filter` only accepts one list, I first zip the `bools` to the `vals` with `cons`, filter on the `bools` with `car`, and then unzip with `cdr` to get the filtered `vals`. 

Throughout my posts on writing Scheme libraries, you will find frequent disclaimers about how my libraries are not written with performance in mind. I simply don't understand Scheme well enough to have good intuition about performance pitfalls. For example, you may have noticed that filtering a dataframe involves zipping the *same* list of boolean values to every column before filtering each column. If dataframes were row based, then the `bools` could be added to the begining of every row with `cons` and the whole dataframe filtered in one pass. That seems obviously better. However, a row-based structure makes extracting columns trickier.

Instead of filtering on every column separately, I could transpose the dataframe to the row-based form, add the `bools`, filter on the whole dataframe at once, remove the `bools`, and transpose back to the column-based form. My guess is that transposing a dataframe to row based and back might generally be faster than zipping and unzipping every column with `bools`, particularly if a dataframe has lots of columns, but I find it easier to think about operating on columns and I was striving for internal consistency across the dataframe procedures. 

### Partition

Because R doesn't allow multiple return values, you would partition a dataframe with two `dplyr::filter` statements. 

```
> dplyr::filter(df, grp == "x")
  trt grp rsp ind
1   a   x   1   0
2   a   x   1   1
3   a   x   1   2
4   b   x   3   6
5   b   x   3   7
6   b   x   3   8

> dplyr::filter(df, grp == "y")
  trt grp rsp ind
1   a   y   2   3
2   a   y   2   4
3   a   y   2   5
4   b   y   4   9
5   b   y   4  10
6   b   y   4  11
```

As mentioned in the [previous post](/split-bind-append-dataframes-scheme/), `dataframe-partition*` allows for partitioning a dataframe based on the `expr`. 

```
> (define-values (keep drop)
    (dataframe-partition* df (grp) (string=? grp "x")))
  
> (dataframe-display keep)

 dim: 6 rows x 4 cols
     trt     grp     rsp     ind 
   <str>   <str>   <num>   <num> 
       a       x      1.      0. 
       a       x      1.      1. 
       a       x      1.      2. 
       b       x      3.      6. 
       b       x      3.      7. 
       b       x      3.      8. 

> (dataframe-display drop)

 dim: 6 rows x 4 cols
     trt     grp     rsp     ind 
   <str>   <str>   <num>   <num> 
       a       y      2.      3. 
       a       y      2.      4. 
       a       y      2.      5. 
       b       y      4.      9. 
       b       y      4.     10. 
       b       y      4.     11. 
```

### Sort

In `dplyr`, dataframes are sorted on multiple columns with `arrange`. By default, columns are sorted in ascending order with `desc` used to indicate descending order. The dataframe is sorted in the order that column names are listed (left to right).

```
> dplyr::arrange(df, grp, desc(ind))
   trt grp rsp ind
1    b   x   3   8
2    b   x   3   7
3    b   x   3   6
4    a   x   1   2
5    a   x   1   1
6    a   x   1   0
7    b   y   4  11
8    b   y   4  10
9    b   y   4   9
10   a   y   2   5
11   a   y   2   4
12   a   y   2   3
```

Similarly, `dataframe-sort*` sorts the dataframe in the order that predicates and column names are listed (left to right) in the `expr` [[1]](#1).

```
> (dataframe-display
   (dataframe-sort* df (string<? grp) (> ind))
   12)
   
 dim: 12 rows x 4 cols
     trt     grp     rsp     ind 
   <str>   <str>   <num>   <num> 
       b       x      3.      8. 
       b       x      3.      7. 
       b       x      3.      6. 
       a       x      1.      2. 
       a       x      1.      1. 
       a       x      1.      0. 
       b       y      4.     11. 
       b       y      4.     10. 
       b       y      4.      9. 
       a       y      2.      5. 
       a       y      2.      4. 
       a       y      2.      3. 
```

#### Implementation

`dataframe-sort*` is a very simple macro.

```
(define-syntax dataframe-sort*
  (syntax-rules ()
    [(_ df (predicate name) ...)
      (df-sort df (list predicate ...) (list (quote name) ...)
              "(dataframe-sort* df (predicate name) ...)")]))
```

The following are equivalent

```
(dataframe-sort* df (string<? grp) (> ind))

(dataframe-sort df (list string<? >) '(grp ind))
```

The justification for this macro is similar to `dataframe-filter*`, i.e., passing predicates and names together reduces likelihood of mixing up the order of the predicate list and the column name list. Does that save enough keystrokes to justify using a macro? It will be interesting to see where I stand on this issue after I become a more experienced Scheme programmer.

Here are the steps involved in `dataframe-sort*`:

1. Extract the columns selected for sorting from the dataframe.
2. Calculate the weights for each column. [The objective is to weight subsequent columns less and less to maintain sorting priority.]  
    a. Pick arbitrary weight for first column.  
    b. Loop through the rest of the columns and set weight of next column to the weight of the previous column divided by the length of unique values in the next column.  
3. Sort unique values in each column by the predicate provided for that column.
4. Enumerate (rank) each unique value and multiply rank by weight.
5. Match weighted ranks for unique values to all values in the column.
6. Sum weighted ranks across columns for each row.
7. Sort all columns in the dataframe by the weighted rank sums.  

`dataframe-sort*` is applied to each column and carries the same potential performance pitfalls described above for `dataframe-filter*`. 

***

<a name="1"></a> [1] I've confused myself several times with respect to how to interpret `<` and `>` in a sort. I guess the intuition is that `<` is ascending because the smaller value is on the left (and vice versa with `>`).