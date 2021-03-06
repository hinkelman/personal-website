+++
title = "A dataframe record type for Chez Scheme"
date = 2020-03-27
updated = 2021-04-26
[taxonomies]
categories = ["dataframe", "Chez Scheme"]
tags = ["dataframe", "data-structures", "association-list"]
+++

As an exercise in my Chez Scheme learning journey, I have implemented a [dataframe record type](https://github.com/hinkelman/dataframe/) and procedures to work with the dataframe record type. Dataframes are column-oriented, tabular data structures useful for data analysis found in several languages including [R](https://stat.ethz.ch/R-manual/R-devel/library/base/html/data.frame.html), [Python](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html), [Julia](https://juliadata.github.io/DataFrames.jl/stable/), and [Go](https://github.com/rocketlaunchr/dataframe-go). In this post, I will introduce the dataframe record type and basic procedures for working with dataframes. In subsequent posts, I will describe other dataframe procedures, e.g., filter, sort, aggregate, etc.

<!-- more -->

A key design decision was to opt for an immutable data structure. Thus, dataframes are based on association lists (alists) rather than hashtables. I find it easier to reason about immutable data and thought the performance trade-off was worth it for this excercise. Here are the properties of a dataframe:

* an alist where each sublist is a column;
* the first element of each column is the column name;
* the column name must be a symbol;
* all column names must be unique;
* all columns must have the same length.

### Dataframe record type

I won't profess to have a good understanding of record types. This is what I came up with for dataframes.

```
(define-record-type dataframe (fields alist names dim)
                    (protocol
                     (lambda (new)
                       (lambda (alist)
                         (let ([proc-string "(make-dataframe alist)"])
                           (check-alist alist proc-string))
                         (new alist
                              (map car alist)
                              (cons (length (cdar alist)) (length alist)))))))
```

A key component of the record definition is `check-alist`, which confirms that the alist meets the definition of a dataframe (see bulleted list above). Each dataframe has three fields (i.e., `alist`, `names`, and `dim`), but `alist` is the only required field. The other two are based on the properties of the `alist`. `define-record-type` creates a predicate, `dataframe?`, constructor procedure, `make-dataframe`, and accessor procedures for each field: `dataframe-alist`, `dataframe-names`, and `dataframe-dim`. 

```
> (define df (make-dataframe '((a 1 2 3) (b 4 5 6))))

> df
#[#{dataframe cziqfonusl4ihl0gdwa8clop7-3} ((a 1 2 3) (b 4 5 6)) (a b) (3 . 2)]

> (dataframe? df)
#t

> (dataframe? '((a 1 2 3) (b 4 5 6)))
#f

> (dataframe-alist df)  
((a 1 2 3) (b 4 5 6))

> (dataframe-names df)
(a b)

> (dataframe-dim df)
(3 . 2)                  ; (rows . columns)

> (define df (make-dataframe '(("a" 1 2 3) ("b" 4 5 6))))
Exception in (make-dataframe alist): names are not symbols

> (dataframe-display df)
 dim: 3 rows x 2 cols
     a     b 
    1.    4. 
    2.    5. 
    3.    6. 
```

### Head and tail

In R, I frequently use `head` to preview the first few rows of a dataframe and, less frequently, use `tail` to view the last few rows. Chez Scheme provides `list-head` and `list-tail` with similar functionality. However, `tail` in R returns the last `n` rows of the dataframe whereas `list-tail` in Chez Scheme returns the rest of the list starting at a given index. My first instinct was to write `dataframe-tail` to use the R behavior, but eventually decided that `dataframe-tail` should follow the behavior established by `list-tail`. I was trying to think in terms of the [principle of least surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment), but the degree of surprise depends on the potential users. Am I targeting R or Scheme programmers? The most realistic scenario is that future me is the only potential user and I want that guy to think in terms of typical Scheme patterns.

```
(define (dataframe-head df n)
  (let ([proc-string  "(dataframe-head df n)"])
    (check-dataframe df proc-string)
    (check-integer-positive n "n" proc-string)
    (check-index n (car (dataframe-dim df)) proc-string)
    (make-dataframe (alist-head-tail (dataframe-alist df) n list-head))))

;; dataframe-tail is based on list-tail, which does not work the same as tail in R
(define (dataframe-tail df n)
  (let ([proc-string  "(dataframe-tail df n)"])
    (check-dataframe df proc-string)
    (check-integer-gte-zero n "n" proc-string)
    (check-index (sub1 n) (car (dataframe-dim df)) proc-string)
    (make-dataframe (alist-head-tail (dataframe-alist df) n list-tail))))

(define (alist-head-tail alist n proc)
  (map (lambda (col) (cons (car col) (proc (cdr col) n))) alist))
```

`dataframe-head` and `dataframe-tail` illustrate a common pattern in the `dataframe` library: extracting the alist, breaking the alist into sublists, working on the sublists, and then rebuilding the alist and dataframe. In the case of `dataframe-head` and `dataframe-tail`, the core logic is so simple that most of the code involves checking inputs. 

### Transpose

Dataframes are a column-oriented data structure. However, the more natural pattern when [reading and writing CSV files](/posts/reading-writing-csv-files-chez-scheme/) is to use a row-oriented list, which I'm calling a `rowtable`. `dataframe->rowtable` and `rowtable->dataframe` allow for switching between row and column orientation.   

```
> (define df (make-dataframe '((a 100 300) (b 4 6) (c 700 900))))

> (dataframe->rowtable df)
((a b c) (100 4 700) (300 6 900))

> (dataframe-display (rowtable->dataframe '((a b c) (1 4 7) (2 5 8) (3 6 9)) #t))
 dim: 3 rows x 3 cols
     a     b     c 
    1.    4.    7. 
    2.    5.    8. 
    3.    6.    9. 

> (dataframe-display (rowtable->dataframe '((1 4 7) (2 5 8) (3 6 9)) #f))
 dim: 3 rows x 3 cols
    V0    V1    V2 
    1.    4.    7. 
    2.    5.    8. 
    3.    6.    9. 

> (rowtable->dataframe '(("a" "b" "c") (1 4 7) (2 5 8) (3 6 9)) #t)
Exception in (make-dataframe alist): names are not symbols
```

### Read and write

If you are working exclusively with dataframes, you can read and write them directly (i.e., without transposing to and from rowtables) with `dataframe-read` and `dataframe-write`. These procedures are straightforward because they are simply reading and writing the alists with `read` and `write`. 

```
(define (dataframe-write df path overwrite?)
  (when (and (file-exists? path) (not overwrite?))
    (assertion-violation path "file already exists"))
  (delete-file path)
  (with-output-to-file path
    (lambda () (write (dataframe-alist df)))))

(define (dataframe-read path)
  (make-dataframe (with-input-from-file path read)))
```

### Extract values

`dataframe-values` returns all the values in a column as a simple list. Following R, I've included `$` as an alias for `dataframe-values`. This procedure is particularly useful when modifying and aggregating dataframes (as I will show in a future blog post). `dataframe-values-unique` returns the unique values from a column. 

```
> (define df (make-dataframe '((a 100 200 300) (b 4 5 6) (c 700 800 900))))

> (dataframe-values df 'b)
(4 5 6)

> ($ df 'b)                 
(4 5 6)

> (map (lambda (name) ($ df name)) '(c a))
((700 800 900) (100 200 300))

> (define df1 (make-dataframe '((x a a b) (y c d e))))

> (dataframe-values-unique df1 'x)
(a b)

> (dataframe-values-unique df1 'y)
(c d e)
```

`dataframe-ref` returns a dataframe based on a list of row indices and, optionally, the selected column names. I did not follow the principle of least surprise here because `dataframe-ref` takes a list of indices rather than a single value as in `list-ref`. For dataframes, the scenario of referencing a single row seemed less likely than a range of rows and I wanted to provide the option to simultaneously select the columns returned. 

```
> (define df (make-dataframe '((grp "a" "a" "b" "b" "b")
                               (trt "a" "b" "a" "b" "b")
                               (adult 1 2 3 4 5)
                               (juv 10 20 30 40 50))))

> (dataframe-display (dataframe-ref df '(0 2 4)))
 dim: 3 rows x 4 cols
   grp   trt  adult   juv 
     a     a     1.   10. 
     b     a     3.   30. 
     b     b     5.   50. 

> (dataframe-display (dataframe-ref df '(0 2 4) 'adult 'juv))
 dim: 3 rows x 2 cols
  adult   juv 
     1.   10. 
     3.   30. 
     5.   50. 
```

