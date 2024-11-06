+++
title = "Reading and writing JSON files in R and Chez Scheme"
date = 2020-03-01
updated = 2024-10-29
[taxonomies]
tags = ["R", "Chez Scheme", "dataframe", "JSON"]
+++

I have [previously written](/reading-writing-json-files-r-racket/) about how to read and write JSON files in R and Racket. In re-reading that old post, I'm struck by how it shows me tinkering without understanding. Now that I have pivoted [from learning Racket to learning Chez Scheme](/exploring-scheme-implementations/), I'm revisiting JSON as a data serialization format and actually reading about JSON instead of just playing with JSON packages. 

<!-- more -->

This [paper](https://arxiv.org/pdf/1403.2805.pdf) on the [`jsonlite` package](https://jeroen.cran.dev/jsonlite/index.html) for R was particularly helpful for improving my understanding. One short section succinctly conveys the most critical ideas.

> The JSON format specifies 4 primitive types (string, number, boolean, null) and two universal structures:   
>> A JSON object: an unordered collection of zero or more name/value pairs, where a name is a string and a value is a string, number, boolean, null, object, or array.  
>> A JSON array: an ordered sequence of zero or more values. 
> 
> Both these structures are heterogeneous; i.e. they are allowed to contain elements of different types. Therefore, the native R realization of these structures is a named list for JSON objects, and unnamed list for JSON arrays. However, in practice a list is an awkward, inefficient type to store and manipulate data in R. Most statistical applications work with (homogeneous) vectors, matrices or data frames. In order to give these data structures a JSON representation, we can define certain special cases of JSON structures which get parsed into other, more specific R types. 

### R

A JSON array is defined by square brackets with elements separated by commas. For example, `[[1.1,2,3],[4,5,6],[7,8,9]]` represents a two-dimensional array comprised of numbers (one of the 4 JSON primitive types). By default, `jsonlite` reads this JSON array as a matrix and coerces all of the numbers to doubles. If `simplifyVector = FALSE`, then `fromJSON` reads the array as an unnamed nested list and coerces each element of the list to the appropriate type, i.e., in this example, only one element of the JSON array is coerced to a double. 

```
> x1 <- fromJSON("[[1.1,2,3],[4,5,6],[7,8,9]]")

> class(x1)
[1] "matrix" "array" 

> typeof(x1)
[1] "double"

> x2 <- fromJSON("[[1.1,2,3],[4,5,6],[7,8,9]]", simplifyVector = FALSE)

> class(x2)
[1] "list"

> typeof(x2)
[1] "list"

> lapply(lapply(x2, "[[", 1), typeof) # apply typeof to first element of each sub-list
[[1]]
[1] "double"
[[2]]
[1] "integer"
[[3]]
[1] "integer"
```

A JSON object is defined by curly brackets containing key-value pairs separated by commas. `fromJSON` reads a JSON object as a named list. However, an array of JSON objects is read as a dataframe. 

```
> fromJSON('{"ID": 1, "Species": "CHN", "Length": 43}')

$ID
[1] 1

$Species
[1] "CHN"

$Length
[1] 43

>> fromJSON('[{"ID": 1, "Species": "CHN", "Length": 43}, 
           {"ID": 2, "Species": "STH", "Length": 131}]')

  ID Species Length
1  1     CHN     43
2  2     STH    131
```

Tabular data can be oriented by rows or columns. The default behavior of `toJSON` uses row-based storage.

>However, unfortunately R is an exception in its preference for column-based storage: most languages, systems, databases, APIs, etc, are optimized for record based operations. For this reason, the conventional way to store and communicate tabular data in JSON seems to almost exclusively row based. 

```
> toJSON(data.frame(ID = c(1, 2), Species = c("CHN", "STH"), Length = c(43, 131)))

[{"ID":1,"Species":"CHN","Length":43},{"ID":2,"Species":"STH","Length":131}] 
```

Setting `dataframe = "columns"`, though, creates a column-based JSON representation. 

```
> toJSON(data.frame(ID = c(1, 2), Species = c("CHN", "STH"), Length = c(43, 131)), 
         dataframe = "columns")

{"ID":[1,2],"Species":["CHN","STH"],"Length":[43,131]} 
```

`fromJSON` and `toJSON` are not perfect inverse functions. For example, a dataframe written in a *column-based format* is read as a named list. 

```
> fromJSON(
    toJSON(data.frame(ID = c(1, 2), Species = c("CHN", "STH"), Length = c(43, 131)), 
           dataframe = "columns"))

$ID
[1] 1 2

$Species
[1] "CHN" "STH"

$Length
[1]  43 131
```

### Chez Scheme

For Chez Scheme, I've been exploring the [`json-tools` library](https://akkuscm.org/packages/json-tools/) for working with JSON files. The `json` library maps JSON arrays to lists and JSON objects to vectors. It maps strings, numbers, and booleans to their Scheme types and maps `null` to the symbol `'null`. Using `(import (json))` provides `json-read` and `json-write`.

```
> (json-read (open-string-input-port "[1.1, 2, true, null]"))

(1.1 2 #t null)
```

The row-based JSON representation of tabular data is read as a list of vectors where each vector represents a row comprised of pairs of with the column name and row value. The column-based representation is read as a vector of lists where the first value of each list is the column name and the other values are the column values.

```
;; row-based
> (json-read
   (open-string-input-port
    "[{\"ID\":1,\"Species\":\"CHN\",\"Length\":43},
      {\"ID\":2,\"Species\":\"STH\",\"Length\":131}]")) 

(#(("ID" . 1) ("Species" . "CHN") ("Length" . 43))
  #(("ID" . 2) ("Species" . "STH") ("Length" . 131)))

;; column-based
> (json-read
   (open-string-input-port
    "{\"ID\":[1,2],\"Species\":[\"CHN\",\"STH\"],\"Length\":[43,131]}"))

#(("ID" 1 2) ("Species" "CHN" "STH") ("Length" 43 131))
```

We can recover the same JSON input with `json-write`.

```
> (json-write
   '(#(("ID" . 1) ("Species" . "CHN") ("Length" . 43))
    #(("ID" . 2) ("Species" . "STH") ("Length" . 131))))

[{"ID": 1, "Species": "CHN", "Length": 43}, {"ID": 2, "Species": "STH", "Length": 131}]

> (json-write '#(("ID" 1 2) ("Species" "CHN" "STH") ("Length" 43 131)))

{"ID": [1, 2], "Species": ["CHN", "STH"], "Length": [43, 131]}
```

However, if our Scheme object includes symbols, they will be converted to strings.

```
> (json-write '#(("ID" 1 2) ("Species" CHN STH) ("Length" 43 131)))

{"ID": [1, 2], "Species": ["CHN", "STH"], "Length": [43, 131]}
```

All of the examples above have involved reading JSON from strings and displaying in the REPL. The following code can be used to read and write JSON files.

```
(define scheme-object (call-with-input-file "example.json" json-read))
(call-with-output-file "example-out.json" (lambda (p) (json-write scheme-object p)))
```
