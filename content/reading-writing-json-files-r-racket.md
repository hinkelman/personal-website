+++
title = "Reading and writing JSON files in R and Racket"
date = 2019-06-14
[taxonomies]
tags = ["R", "Racket", "dataframe"]
+++

In learning about [reading CSV files in Racket](/post/reading-csv-files-in-r-and-racket/), I have started to reconsider whether storing small(ish) datasets in CSV files is the best default behavior [[1]](#1). My default was set by primarily working in R, where reading and writing CSV files plays a central role in data analysis. When working solely in R, I expect that my old habits will die hard and CSV files will continue to play a prominent role. However, when passing small(ish) data between R and Racket, I think JSON might be a better alternative [[2]](#2).

<!-- more -->

Before we jump into a bunch of code examples, let's define some wrapper functions to make the subsequent code a bit more concise. In R, we are using the [`jsonlite` package](https://github.com/jeroen/jsonlite) and a little function to write the JSON object to file.

```
library(jsonlite)

write_json <- function(object, filename){
  writeLines(toJSON(object), filename)
}
```

In Racket, we are using the [`json` module](https://docs.racket-lang.org/json/index.html?q=json) and have simple wrapper functions for both reading and writing JSON. A [`jsexpr`](https://docs.racket-lang.org/json/index.html?q=jsexpr#%28tech._jsexpr%29) is a subset of Racket values (e.g., list, hash table, etc.) that can be represented as JSON strings.

```
#lang racket

(require json)

(define (read-json-wrapper filename)
  (call-with-input-file filename read-json))

(define (write-json-wrapper jsexpr filename)
  (call-with-output-file filename
    (λ (x) (write-json jsexpr x))
    #:exists 'replace))
```

### R Data Structures

#### Data Frame

The data frame is the central data structure in R, which corresponds well to flat, delimited text files (in contrast to JSON). In this code chunk, we use the same [example CSV file](/data/example.csv) as in the [CSV reading post](/post/reading-csv-files-in-r-and-racket/). 

```
> write_json(head(read.csv("example.csv")), "R_dataframe.json")
> fromJSON("R_dataframe.json")
        date integer         float  bool char      word yn
1 12/25/2060 -958838  590131109036  true    u      zosu  Y
2 02/18/2023 -165090  528052918839  true    1        em  Y
3 11/27/2064 -207296   99397538939 false    w    doptah  Y
4 06/28/2044  316236  590216172038 false    x gutijenel  N
5 08/12/2045  722897 -309360363517  true    7      hueh  N
6 07/23/1904 -903116 -509332808532  true    0   fufcora  Y
```

Racket reads the data frame as a list of [hash tables](https://docs.racket-lang.org/reference/hashtables.html?q=hasheq#%28def._%28%28quote._~23~25kernel%29._hasheq%29%29). Note that keys (e.g., `bool`, `char`, `date`, etc.) in the hash table are listed in alphabetical order. Also, the redundancy of the JSON format is evident for this tabular data via the repetition of the column names.

```
> (read-json-wrapper "R_dataframe.json")
'(#hasheq((bool . "true")
          (char . "u")
          (date . "12/25/2060")
          (float . 590131109036.032)
          (integer . -958838)
          (word . "zosu")
          (yn . "Y"))
  #hasheq((bool . "true")
          (char . "1")
          (date . "02/18/2023")
          (float . 528052918838.886)
          (integer . -165090)
          (word . "em")
          (yn . "Y"))
  #hasheq((bool . "false")
          (char . "w")
          (date . "11/27/2064")
          (float . 99397538938.88)
          (integer . -207296)
          (word . "doptah")
          (yn . "Y"))
  #hasheq((bool . "false")
          (char . "x")
          (date . "06/28/2044")
          (float . 590216172037.734)
          (integer . 316236)
          (word . "gutijenel")
          (yn . "N"))
  #hasheq((bool . "true")
          (char . "7")
          (date . "08/12/2045")
          (float . -309360363516.723)
          (integer . 722897)
          (word . "hueh")
          (yn . "N"))
  #hasheq((bool . "true")
          (char . "0")
          (date . "07/23/1904")
          (float . -509332808531.968)
          (integer . -903116)
          (word . "fufcora")
          (yn . "Y")))
```

#### Vector

It is unlikely that I would want to write a vector to file, but it is straightforward with `jsonlite`.

```
> write_json(vector(mode = "numeric", length = 10), "R_vector.json")
> fromJSON("R_vector.json")
 [1] 0 0 0 0 0 0 0 0 0 0
```

Racket reads the vector as a list.

```
> (read-json-wrapper "R_vector.json")
'(0 0 0 0 0 0 0 0 0 0)
```

#### Matrix/Array 

The `jsonlite` package handles both matrices and arrays.

```
> write_json(matrix(data = runif(25), nrow = 5, ncol = 5), "R_matrix.json")
> fromJSON("R_matrix.json")
       [,1]   [,2]   [,3]   [,4]   [,5]
[1,] 0.1451 0.3178 0.3240 0.9761 0.2404
[2,] 0.9971 0.1716 0.3767 0.1735 0.7416
[3,] 0.5081 0.7600 0.6993 0.2717 0.1228
[4,] 0.2520 0.5952 0.1561 0.4297 0.4912
[5,] 0.6209 0.8862 0.4749 0.4307 0.2534
```

Racket reads a matrix as a list of lists. It can also read multidimensional arrays by further nesting of lists.

```
> (read-json-wrapper "R_matrix.json")
'((0.1451 0.3178 0.324 0.9761 0.2404)
  (0.9971 0.1716 0.3767 0.1735 0.7416)
  (0.5081 0.76 0.6993 0.2717 0.1228)
  (0.252 0.5952 0.1561 0.4297 0.4912)
  (0.6209 0.8862 0.4749 0.4307 0.2534))
```

#### List

JSON is a hierarchical data format that is well suited for nested lists.

```
> write_json(list("A" = list("One" = 1, 
+                            "Two" = 2,
+                            "Three" = 3),
+                 "B" = list("Four" = 4, 
+                            "Five" = 5,
+                            "Six" = 6)),
+            "R_list.json")
> fromJSON("R_list.json")
$A
$A$One
[1] 1

$A$Two
[1] 2

$A$Three
[1] 3


$B
$B$Four
[1] 4

$B$Five
[1] 5

$B$Six
[1] 6
```

Racket reads the nested list as a nested hash table [[3]](#3) rather than a list of hash tables as with the data frame. As with reading the data frame above, the keys are listed in alphabetical order.

```
> (read-json-wrapper "R_list.json")
'#hasheq((A . #hasheq((One . (1)) (Three . (3)) (Two . (2))))
         (B . #hasheq((Five . (5)) (Four . (4)) (Six . (6)))))
```

### Racket Data Structures

#### List

In this first example, I've just thrown together a bunch of the different Racket values that are valid as `jsexpr`. 

```
> (write-json-wrapper (list 1 2.1 #t "B" "word" (list 3 4) (hash 'C 5 'D '(6 -7))) "Racket_list.json")
> (read-json-wrapper "Racket_list.json")
'(1 2.1 #t "B" "word" (3 4) #hasheq((C . 5) (D . (6 -7))))
```

The `jsonlite` package reads the Racket list as an R list. Note that the boolean value is automatically converted to a logical value in R (`TRUE`). Also, the keys (`C`, `D`) in the hash table are used as names in the nested list.

```
> fromJSON("Racket_list.json")
[[1]]
[1] 1

[[2]]
[1] 2.1

[[3]]
[1] TRUE

[[4]]
[1] "B"

[[5]]
[1] "word"

[[6]]
[1] 3 4

[[7]]
[[7]]$D
[1]  6 -7

[[7]]$C
[1] 5
```

#### Hash Table

If you've read this far, there should be no surprise about how suitable JSON is for a nested hash table. 

```
> (write-json-wrapper (hasheq 'A (hasheq 'C 2 'E 6) 'B (hasheq 'D 4 'F 8)) "Racket_hash.json")
> (read-json-wrapper "Racket_hash.json")
'#hasheq((A . #hasheq((C . 2) (E . 6))) (B . #hasheq((D . 4) (F . 8))))
```

The nested hash table becomes a nested (and named) list in R.

```
> fromJSON("Racket_hash.json")
$A
$A$E
[1] 6

$A$C
[1] 2


$B
$B$D
[1] 4

$B$F
[1] 8
```

#### Table

This table is actually a list of lists where the first row of the list contains the headers for the columns.

```
> (write-json-wrapper (list '("A" "B" "C") '(1 2 3) '(4 5 6)) "Racket_table.json")
> (read-json-wrapper "Racket_table.json")
'(("A" "B" "C") (1 2 3) (4 5 6))
```

`jsonlite` reads this "table" as a matrix and converts all elements to characters because all elements of an R matrix need to be the same type.

```
> fromJSON("Racket_table.json")
     [,1] [,2] [,3]
[1,] "A"  "B"  "C" 
[2,] "1"  "2"  "3" 
[3,] "4"  "5"  "6" 
```

#### List of Hash Tables

A list of hash tables was how the `json` module read the JSON file created by `jsonlite` for a data frame.

```
> (write-json-wrapper (read-json-wrapper "R_dataframe.json") "Racket_list_hash.json")
> (read-json-wrapper "Racket_list_hash.json")
'(#hasheq((bool . "true")
          (char . "u")
          (date . "12/25/2060")
          (float . 590131109036.032)
          (integer . -958838)
          (word . "zosu")
          (yn . "Y"))
  #hasheq((bool . "true")
          (char . "1")
          (date . "02/18/2023")
          (float . 528052918838.886)
          (integer . -165090)
          (word . "em")
          (yn . "Y"))
  #hasheq((bool . "false")
          (char . "w")
          (date . "11/27/2064")
          (float . 99397538938.88)
          (integer . -207296)
          (word . "doptah")
          (yn . "Y"))
  #hasheq((bool . "false")
          (char . "x")
          (date . "06/28/2044")
          (float . 590216172037.734)
          (integer . 316236)
          (word . "gutijenel")
          (yn . "N"))
  #hasheq((bool . "true")
          (char . "7")
          (date . "08/12/2045")
          (float . -309360363516.723)
          (integer . 722897)
          (word . "hueh")
          (yn . "N"))
  #hasheq((bool . "true")
          (char . "0")
          (date . "07/23/1904")
          (float . -509332808531.968)
          (integer . -903116)
          (word . "fufcora")
          (yn . "Y")))
```

Apparently, the `json` module keeps track of the original order of the keys (even though they are sorted in the hash table) because the order of the columns in the data frame is preserved.

```
> fromJSON("Racket_list_hash.json")
        date  bool integer      word yn char         float
1 12/25/2060  true -958838      zosu  Y    u  590131109036
2 02/18/2023  true -165090        em  Y    1  528052918839
3 11/27/2064 false -207296    doptah  Y    w   99397538939
4 06/28/2044 false  316236 gutijenel  N    x  590216172038
5 08/12/2045  true  722897      hueh  N    7 -309360363517
6 07/23/1904  true -903116   fufcora  Y    0 -509332808532
```

### Conclusion

My primary use case for JSON files would be to create lookup tables in R for input values that are used in simulation models built in Racket. In R, I have tended to use data frames for lookup tables because a data frame is the typical output of a data analysis pipeline. I will need to learn more about working with lists of hash tables and nested hash tables in Racket to decide if it makes sense to reformat a data frame into a nested list before writing the object to JSON. 

***

<a name="1"></a> [1] I even used CSV files as a [database replacement](/post/dt-datatable-crud/) in my [Shiny Scorekeeper app](https://github.com/hinkelman/Shiny-Scorekeeper). 

<a name="2"></a> [2] See also [this post](/posts/reading-and-writing-json-files-in-r-and-chez-scheme/) for more on reading and writing JSON files.

<a name="3"></a> [3] See also [previous post](/post/named-lists-hash-tables/) on the similarity of nested lists in R and nested hash tables in Racket.