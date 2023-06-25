+++
title =  "Split, bind, and append dataframes in Chez Scheme"
date = 2020-04-04
updated = 2023-06-24
[taxonomies]
categories = ["dataframe", "Chez Scheme"]
tags = ["dataframe", "data-structures", "association-list", "replicate", "rep", "cbind", "dplyr", "bind_rows"]
+++

This post is the third in a [series](/categories/dataframe/) on the [`dataframe` library](https://github.com/hinkelman/dataframe/) for Chez Scheme. In this post, I will contrast the `dataframe` library with functions from base R and the [`dplyr` package](https://dplyr.tidyverse.org) for splitting, binding, and appending dataframes.

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
   (list (cons 'trt (append (make-list 6 "a") (make-list 6 "b")))
         (cons 'grp (append (make-list 3 "x") (make-list 3 "y")
                            (make-list 3 "x") (make-list 3 "y")))
         (cons 'rsp (append (make-list 3 1) (make-list 3 2)
                            (make-list 3 3) (make-list 3 4))))))

(define df2
  (make-dataframe
   (list (cons 'asc (iota 12))
         (cons 'desc (reverse (iota 12))))))
```

I think the Scheme code for creating `df1` is clear, but I like the conciseness of the R code. I'm taking this post on a little detour to write a `rep` procedure to replicate (pun intended) the functionality of `rep`. 

```
(define (rep ls n type)
  (cond [(symbol=? type 'each)
         (apply append (map (lambda (x) (make-list n x)) ls))]
        [(symbol=? type 'times)
         (rep-times ls n)]
        [else
         (assertion-violation "(rep ls n type)"
                              "type must be 'each or 'times")]))

(define (rep-times ls n)
  (define (loop ls-out n)
    (if (= n 1) ls-out (loop (append ls ls-out) (sub1 n))))
  (loop ls n))

(define df1
  (make-dataframe
   (list (cons 'trt (rep '("a" "b") 6 'each))
         (cons 'grp (rep (rep '("x" "y") 3 'each) 2 'times))
         (cons 'rsp (rep '(1 2 3 4) 3 'each)))))
```

The `each` case was a simple `map`, but I couldn't think of a way to use higher-order functions for the `times` case. Instead, I wrote a separate recursive procedure, `rep-times`, to handle that case.

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

In Chez Scheme, we append dataframes with equal numbers of rows via `dataframe-append`.

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

I chose `dataframe-append` as the name because alists, which are at the heart of dataframes, are straightforwardly combined with `append` in Chez Scheme.

```
> (append '((a 1 2 3)) '((b 4 5 6)))
((a 1 2 3) (b 4 5 6))
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
   trt   grp   rsp 
     b     x    3. 
     b     x    3. 
     b     x    3. 
 dim: 3 rows x 3 cols
   trt   grp   rsp 
     b     y    4. 
     b     y    4. 
     b     y    4. 
 dim: 3 rows x 3 cols
   trt   grp   rsp 
     a     x    1. 
     a     x    1. 
     a     x    1. 
 dim: 3 rows x 3 cols
   trt   grp   rsp 
     a     y    2. 
     a     y    2. 
     a     y    2. 
```

#### Implementation

The first step in `dataframe-split` is to find the unique values of the grouping columns [[1]](#1).

```
> (dataframe-display (dataframe-unique (dataframe-select df1 'trt 'grp)))
 dim: 4 rows x 2 cols
   trt   grp 
     a     x 
     a     y 
     b     x 
     b     y 
```

`dataframe-unique` involves transposing the alist to a row-based structure to remove duplicates and then transposing back to the column-based structure. This is another example of me choosing a straightforward solution over an efficient one [[2]](#2).

```
(define (transpose ls) (apply map list ls))

;; https://stackoverflow.com/questions/8382296/scheme-remove-duplicated-numbers-from-list
(define (remove-duplicates ls)
  (cond [(null? ls)
         '()]
        [(member (car ls) (cdr ls))
         (remove-duplicates (cdr ls))]
        [else
         (cons (car ls) (remove-duplicates (cdr ls)))]))
         
> (remove-duplicates '((trt "a" "a" "a") (grp "x" "x" "x")))
((trt "a" "a" "a") (grp "x" "x" "x"))

> (transpose '((trt "a" "a" "a") (grp "x" "x" "x")))
((trt grp) ("a" "x") ("a" "x") ("a" "x"))

> (remove-duplicates (cdr (transpose '((trt "a" "a" "a") (grp "x" "x" "x")))))
(("a" "x"))
```

We loop through the rows of the unique groups and partition the dataframe [[3]](#3). `dataframe-partition*` returns two dataframes. The `keep` and `drop` dataframes contain the rows where the `expr` is `#t` and `#f`, respectively. I'm using `*` to indicate that this is a macro. There is a more verbose option without the `*` in the name.

```
> (define-values (keep drop)
    (dataframe-partition*
     df1 (trt grp) (and (string=? trt "a") (string=? grp "x"))))
     
> (dataframe-display keep)
  dim: 3 rows x 3 cols
   trt   grp   rsp 
     a     x    1. 
     a     x    1. 
     a     x    1. 

> (dataframe-display drop)
 dim: 9 rows x 3 cols
   trt   grp   rsp 
     a     y    2. 
     a     y    2. 
     a     y    2. 
     b     x    3. 
     b     x    3. 
     b     x    3. 
     b     y    4. 
     b     y    4. 
     b     y    4. 
```

The `keep` dataframe becomes the first dataframe in the list of dataframes returned by `dataframe-split`. The algorithm continues looping through the rows of unique groups and partitions the `drop` dataframe in each subsequent iteration. 

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

`dataframe-bind` works similarly to `dplyr::bind_rows`, but we need to use `apply` to bust open the list of dataframes created by `dataframe-split`. 

```
> (dataframe-display (apply dataframe-bind (dataframe-split df1 'trt 'grp)) 12)
 dim: 12 rows x 3 cols
   trt   grp   rsp 
     a     x    1. 
     a     x    1. 
     a     x    1. 
     a     y    2. 
     a     y    2. 
     a     y    2. 
     b     x    3. 
     b     x    3. 
     b     x    3. 
     b     y    4. 
     b     y    4. 
     b     y    4. 
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

Because Chez Scheme doesn't have explicit missing values, I created a separate procedure, `dataframe-bind-all`, for binding dataframes where missing columns are filled by the specified missing value.

```
> (dataframe-display (dataframe-bind-all -999 df-a (dataframe-drop df-b 'rsp)) 12)
 dim: 12 rows x 3 cols
   trt   grp    rsp 
     a     x     1. 
     a     x     1. 
     a     x     1. 
     a     y     2. 
     a     y     2. 
     a     y     2. 
     b     x  -999. 
     b     x  -999. 
     b     x  -999. 
     b     y  -999. 
     b     y  -999. 
     b     y  -999. 
```

In contrast, `dataframe-bind` will drop all columns not shared across the dataframes being bound.

```
> (dataframe-display (dataframe-bind df-a (dataframe-drop df-b 'rsp)) 12)
 dim: 12 rows x 2 cols
   trt   grp 
     a     x 
     a     x 
     a     x 
     a     y 
     a     y 
     a     y 
     b     x 
     b     x 
     b     x 
     b     y 
     b     y 
     b     y 
```

### Final thoughts

With the exception of `dataframe-split`, all of the procedures described in the first three posts in the [dataframe series](/categories/dataframe/) involve straightforward composition of Scheme's fundamental procedures (e.g., `map`, `apply`, `append`, `cons`, `car`, `cdr`, etc.) on Scheme's core data structure, i.e., lists. The next couple of posts involve procedures that forced me to wrestle with tradeoffs between convenient syntax via macros (e.g., `dataframe-partition*`) and familiarity/consistency with Chez Scheme's standard library. In the next post, I will describe how to filter, partition, and sort dataframes in Chez Scheme. 

***

<a name="1"></a> [1] I'm illustrating the ideas with the user-facing `dataframe` procedures, but inside `dataframe-split`, and most `dataframe` procedures, are procedures that work directly on alists to avoid the overhead of unwrapping and rewrapping the alists as dataframes. 

<a name="2"></a> [2] This is not to say that I know a more efficient solution, but, rather, that I opted for a straightforward solution even though it contains the (significant?) overhead of transposing the list a couple of times.

<a name="3"></a> [3] `dataframe-partition` will be covered in the next post of this series.