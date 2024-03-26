+++
title = "Data transformation in Scheme"
date = 2023-07-07
updated = 2024-03-26
[taxonomies]
categories = ["Scheme", "Chez Scheme", "dataframe"]
tags = ["R4DS", "dplyr"]
+++

I have done some recent work on my [`dataframe`](https://github.com/hinkelman/dataframe/) library for Scheme (R6RS) and thought I would run through the examples in the [Data Transformation chapter](https://r4ds.hadley.nz/data-transform.html) of [R for Data Science](https://r4ds.hadley.nz/) (R4DS). In this post, I won't reproduce any of the R code and will provide limited commentary on the Scheme code (which is also available via this [gist](https://gist.github.com/hinkelman/0945b3c905dcd244809bbed81d2faeb1)).

<!-- more -->

### Setup

The `nycflights13::flights` dataset is used for all of the examples shown below. I've written it to a file and posted it [here](/data/nycflights.tsv).

```
(import (dataframe))

(define flights (tsv->dataframe "nycflights.tsv"))
```
The `flights` dataset has 336,776 rows and 19 columns. Datasets of this size strain the dataframe library and provide a suboptimal experience, especially compared to R. Skipped sections indicate that the `dataframe` library has no equivalent capabilities. 

### 3.1 Introduction

*3.1.3*

```
(-> flights
    (dataframe-filter* (dest) (string=? dest "IAH")) 
    (dataframe-aggregate*
     (year month day)
     (arr_delay (arr_delay) (inexact (mean arr_delay))))
    (dataframe-display))
```

### 3.2 Rows

*3.2.1*

Piping all of the output to a `dataframe-display` will often not show clearly the operation was successful. The one liners below show that the operation yielded the expected result. Need to remove `'na` from the `dep_delay` column or the `>` operation will fail.

```
(define delayed-flights
  (-> flights
      (dataframe-filter*
       (dep_delay)
       (and (not (na? dep_delay))
            (> dep_delay 120)))))

(apply min ($ delayed-flights 'dep_delay))

(-> flights
    (dataframe-filter*
     (month day)
     (and (not (or (na? month) (na? day)))
          (= month 1)
          (= day 1)))
    (dataframe-glimpse))

(define jan-feb-flights
  (-> flights
      (dataframe-filter*
       (month)
       (and (not (na? month))
            (or (= month 1)
                (= month 2))))))

(remove-duplicates ($ jan-feb-flights 'month))

(-> flights
    (dataframe-filter*
     (month)
     (and (not (na? month))
          (member month '(1 2))))
     (dataframe-glimpse))
```

*3.2.3*

```
(-> flights
    (dataframe-filter*
     (year month day dep_time)
     (not (or (na? year) (na? month) (na? day) (na? dep_time))))
    (dataframe-sort*
     (< year) (< month) (< day) (< dep_time))
    (dataframe-glimpse))


(-> flights
    (dataframe-filter*
     (dep_delay)
     (not (na? dep_delay)))
    (dataframe-sort*
     (> dep_delay))
    (dataframe-glimpse))
```

*3.2.4*

`dplyr::distinct` has a parameter, `.keep_all`, that keeps all columns for the first occurrence of each unique combo. `dataframe-unique` does not have that functionality.

```
(-> flights
    (dataframe-unique)
    (dataframe-glimpse))

(-> flights
    (dataframe-select* origin dest)
    (dataframe-unique)
    (dataframe-glimpse))
```

The `dataframe` library does not include a procedure comparable to `dplyr::count`, but the same result can be obtained with `dataframe-aggregate*`.

```
(-> flights
    (dataframe-aggregate*
     (origin dest)
     (n (origin) (length origin)))
    (dataframe-sort*
     (> n))
    (dataframe-display))
```

### 3.3 Columns

*3.3.1*

```
(-> flights
    (dataframe-filter*
     (dep_delay arr_delay distance air_time)
     (not (or (na? dep_delay) (na? arr_delay) (na? distance) (na? air_time))))
    (dataframe-modify*
     (gain (dep_delay arr_delay) (- dep_delay arr_delay))
     (speed (distance air_time) (* (/ distance air_time) 60)))
    (dataframe-glimpse))
```

`dataframe-modify*` has no equivalent to the `.before` and `.after` parameters of `dplyr::mutate`. Using a `dataframe-select` to get focal columns but this drops all other columns.

```
(-> flights
    (dataframe-filter*
     (dep_delay arr_delay distance air_time)
     (not (or (na? dep_delay) (na? arr_delay) (na? distance) (na? air_time))))
    (dataframe-modify*
     (gain (dep_delay arr_delay) (- dep_delay arr_delay))
     (speed (distance air_time) (inexact (* (/ distance air_time) 60))))
    (dataframe-select* year month day gain speed)
    (dataframe-display))
```

*3.3.2*

`dplyr::select` has many wonderful helper functions to do complicated select operations easily. `dataframe-select*` is very simple.

```
(-> flights
    (dataframe-select* year month day)
    (dataframe-display))
```

*3.3.3*
```
(-> flights
    (dataframe-rename* (tailnum tail_num))
    (dataframe-glimpse))
```

### 3.4 The Pipe

```
(-> flights
    (dataframe-filter*
     (dest distance air_time)
     (and (not (or (na? dest) (na? distance) (na? air_time)))
          (string=? dest "IAH")))
    (dataframe-modify*
     (speed
      (distance air_time)
      (inexact (* (/ distance air_time) 60))))
    (dataframe-select*
     year month day dep_time carrier flight speed)
    (dataframe-sort*
     (> speed))
    (dataframe-display))

(dataframe-display
 (dataframe-sort*
  (dataframe-select*
   (dataframe-modify*
    (dataframe-filter*
     flights
     (dest distance air_time)
     (and (not (or (na? dest) (na? distance) (na? air_time)))
          (string=? dest "IAH")))
    (speed
     (distance air_time)
     (inexact (* (/ distance air_time) 60))))
   year month day dep_time carrier flight speed)
  (> speed)))

(define flights1
  (dataframe-filter*
   flights
   (dest distance air_time)
   (and (not (or (na? dest) (na? distance) (na? air_time)))
        (string=? dest "IAH"))))

(define flights2
  (dataframe-modify*
   flights1
   (speed
    (distance air_time)
    (inexact (* (/ distance air_time) 60)))))

(define flights3
  (dataframe-select*
   flights2
   year month day dep_time carrier flight speed))

(dataframe-display
 (dataframe-sort* flights3 (> speed)))
```

### 3.5 Groups

*3.5.2*

`dplyr::group_by` automatically sorts by grouping variables, but that step needs to be done explicitly in `dataframe-aggregate*`.

```
(-> flights
    (dataframe-aggregate*
     (month)
     (avg_delay (dep_delay) (inexact (mean dep_delay))))
    (dataframe-sort*
     (< month))
    (dataframe-display))

(-> flights
    (dataframe-filter*
     (dep_delay)
     (not (na? dep_delay)))
    (dataframe-aggregate*
     (month)
     (avg_delay (dep_delay) (inexact (mean dep_delay)))
     (n (dep_delay) (length dep_delay)))
    (dataframe-sort*
     (< month))
    (dataframe-display))
```

*4.5.5*

`dataframe-aggregate*` struggles when splitting a large dataframe into many groups (in this case, 365). The operation to summarize monthly results (not shown) is reasonably quick. In this case, I didn't bother to filter because I knew there were no missing values for `year`, `month`, and `day`.

```
(define daily_flights
  (dataframe-aggregate*
   flights
   (year month day)
   (n (dep_time) (length dep_time))))
```
