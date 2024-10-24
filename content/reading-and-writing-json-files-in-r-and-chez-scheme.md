+++
title = "Reading and writing JSON files in R and Chez Scheme"
date = 2020-03-01
[taxonomies]
tags = ["R", "Chez Scheme", "dataframe"]
+++

I have [previously written](/reading-writing-json-files-r-racket/) about how to read and write JSON files in R and Racket. In re-reading that old post, I'm struck by how it shows me tinkering without understanding. Now that I have pivoted [from learning Racket to learning Chez Scheme](/exploring-scheme-implementations/), I'm revisiting JSON as a data serialization format and actually reading about JSON instead of just playing with JSON packages. 

<!-- more -->

This [paper](https://arxiv.org/pdf/1403.2805.pdf) on the [`jsonlite` package](https://jeroen.cran.dev/jsonlite/index.html) for R was particularly helpful for improving my understanding. One short section succinctly conveys the most critical ideas.

> The JSON format specifies 4 primitive types (string, number, boolean, null) and two universal structures: 
>
> * A JSON object: an unordered collection of zero or more name/value pairs, where a name is a string and
a value is a string, number, boolean, null, object, or array.
> * A JSON array: an ordered sequence of zero or more values.
>
> Both these structures are heterogeneous; i.e. they are allowed to contain elements of different types. Therefore, the native R realization of these structures is a named list for JSON objects, and unnamed list for JSON arrays. However, in practice a list is an awkward, inefficient type to store and manipulate data in R. Most statistical applications work with (homogeneous) vectors, matrices or data frames. In order to give these data structures a JSON representation, we can define certain special cases of JSON structures which get parsed into other, more specific R types. 

### R

A JSON array is defined by square brackets with elements separated by commas. For example, `[[1.1,2,3],[4,5,6],[7,8,9]]` represents a two-dimensional array comprised of numbers (one of the 4 JSON primitive types). By default, `jsonlite` reads this JSON array as a matrix and coerces all of the numbers to doubles. If `simplifyVector = FALSE`, then `fromJSON` reads the array as an unnamed nested list and coerces each element of the list to the appropriate type, i.e., in this example, only one element of the JSON array is coerced to a double. 

```
> x1 <- fromJSON("[[1.1,2,3],[4,5,6],[7,8,9]]")
> class(x1)
[1] "matrix"
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

> fromJSON('[{"ID": 1, "Species": "CHN", "Length": 43}, {"ID": 2, "Species": "STH", "Length": 131}]')
  ID Species Length
1  1     CHN     43
2  2     STH    131
```

Tabular data can be oriented by rows or columns. The default behavior of `fromJSON` uses row-based storage.

>However, unfortunately R is an exception in its preference for column-based storage: most languages, systems, databases, APIs, etc, are optimized for record based operations. For this reason, the conventional way to store and communicate tabular data in JSON seems to almost exclusively row based. 

```
> toJSON(data.frame(ID = c(1, 2), Species = c("CHN", "STH"), Length = c(43, 131)))
[{"ID":1,"Species":"CHN","Length":43},{"ID":2,"Species":"STH","Length":131}] 
```

Setting `dataframe = "columns"`, though, creates a column-based JSON representation. 

```
> toJSON(data.frame(ID = c(1, 2), Species = c("CHN", "STH"), Length = c(43, 131)), dataframe = "columns")
{"ID":[1,2],"Species":["CHN","STH"],"Length":[43,131]} 
```

`fromJSON` and `toJSON` are not perfect inverse functions. For example, a dataframe written in a columnar format is read as a named list. 

```
> fromJSON(toJSON(data.frame(ID = c(1, 2), Species = c("CHN", "STH"), Length = c(43, 131)), dataframe = "columns"))
$ID
[1] 1 2

$Species
[1] "CHN" "STH"

$Length
[1]  43 131
```

### Chez Scheme

For Chez Scheme, I've been exploring the [`json` library](https://guenchi.github.io/json/) for working with JSON files. The `json` library maps JSON arrays to vectors and JSON objects to association lists. `null`, `true`, and `false` are mapped as symbols rather than `'()`, `#t`, and `#f`. 

```
> (define x (string->json "[1.1 2 true]"))
> x
#(1.1 2 true)
> (integer? (vector-ref x 0))
#f
> (integer? (vector-ref x 1))
#t
> (symbol? (vector-ref x 2))
#t
```

The row-based JSON representation of tabular data is read as a vector of association lists where each association list represents one row. The column-based representation is read as an association list where each key is a column name and the values are a vector.

```
> (string->json "[{ID: 1, Species: CHN, Length: 43}, {ID: 2, Species: STH, Length: 131}]")   ; row based
#(((ID . 1) (Species . CHN) (Length . 43))
  ((ID . 2) (Species . STH) (Length . 131)))
  
> (string->json "{ID:[1,2], Species:[CHN,STH], Length:[43,131]}")                            ; column based
((ID . #(1 2)) (Species . #(CHN STH)) (Length . #(43 131)))
```

The above examples are not valid JSON, which `string->json` doesn't enforce. However, `json->string` requires a valid JSON represention, which, in this example, involves using strings not symbols.

```
> (json->string '((ID . #(1 2)) (Species . #(CHN STH)) (Length . #(43 131))))
Exception in string-append: ID is not a string

> (display (json->string '(("ID" . #(1 2)) ("Species" . #("CHN" "STH")) ("Length" . #(43 131)))))
{"ID":[1,2],"Species":["CHN","STH"],"Length":[43,131]}
```

What the `json` library lacks in parsing functionality, it makes up for in tools for working with JSON data structures. The `json` library provides example JSON data that when parsed by `string->json` creates the following object, `x`.

```
#((("Number" . 1) ("Name" . "Laetetia") ("Gender" . "female") ("Age" . 16)
    ("Father"
      ("Number" . 2)
      ("Name" . "Louis")
      ("Age" . 48)
      ("Revenue" . 1000000))
    ("Mother"
      ("Number" . 3)
      ("Name" . "Lamia")
      ("Age" . 43)
      ("Revenue" . 800000))
    ("Revenue" . 100000)
    ("Score"
      ("Math" ("School" . 8) ("Exam" . 9))
      ("Literature" ("School" . 9) ("Exam" . 9))))
  (("Number" . 4) ("Name" . "Tania") ("Gender" . "female") ("Age" . 17)
    ("Father"
      ("Number" . 5)
      ("Name" . "Thomas")
      ("Age" . 45)
      ("Revenue" . 150000))
    ("Mother"
      ("Number" . 6)
      ("Name" . "Jenney")
      ("Age" . 42)
      ("Revenue" . 180000))
    ("Revenue" . 80000)
    ("Score"
      ("Math" ("School" . 7) ("Exam" . 8))
      ("Literature" ("School" . 10) ("Exam" . 6))))
  (("Number" . 7) ("Name" . "Anne") ("Gender" . "female") ("Age" . 18)
    ("Father"
      ("Number" . 8)
      ("Name" . "Alex")
      ("Age" . 40)
      ("Revenue" . 200000))
    ("Mother"
      ("Number" . 9)
      ("Name" . "Sicie")
      ("Age" . 43)
      ("Revenue" . 50000))
    ("Revenue" . 120000)
    ("Score"
      ("Math" ("School" . 8) ("Exam" . 8))
      ("Literature" ("School" . 6) ("Exam" . 8)))))
```

`json-ref` provides shorthand for extracting pieces of the data structure. The 1st argument to `json-ref` is the data structure, `x`, the 2nd argument is a numeric index because we are working with a vector of association lists, and the remaining arguments are the keys listed in the order of nesting in the data structure.

```
> (json-ref x 2 "Name")
"Anne"
> (json-ref x 2 "Father")
(("Number" . 8)
  ("Name" . "Alex")
  ("Age" . 40)
  ("Revenue" . 200000))
> (json-ref x 2 "Score" "Math" "Exam")
8
```

`json-set` allows for changing elements of the JSON representation. In this example, we rescale the scores from a scale of 0-20 to 0-100. Importantly, `json-set` is returning a new object, not modifying `x`. This example also illustrates the use of `#t` to indicate that the new values are applied to all positions at the specified depth in the data structure, e.g., first `#t` means use all elements of the vector, the next `#t` means use all of the math scores, and the final `#t` means use all of the literature scores. The procedure is then applied to all the values identified by the `#t`'s and keys.

```
> (json-set x #t "Score" #t #t (lambda (x) (* x 5)))
#((("Number" . 1) ("Name" . "Laetetia") ("Gender" . "female") ("Age" . 16)
    ("Father"
      ("Number" . 2)
      ("Name" . "Louis")
      ("Age" . 48)
      ("Revenue" . 1000000))
    ("Mother"
      ("Number" . 3)
      ("Name" . "Lamia")
      ("Age" . 43)
      ("Revenue" . 800000))
    ("Revenue" . 100000)
    ("Score"
      ("Math" ("School" . 40) ("Exam" . 45))
      ("Literature" ("School" . 45) ("Exam" . 45))))
  (("Number" . 4) ("Name" . "Tania") ("Gender" . "female") ("Age" . 17)
    ("Father"
      ("Number" . 5)
      ("Name" . "Thomas")
      ("Age" . 45)
      ("Revenue" . 150000))
    ("Mother"
      ("Number" . 6)
      ("Name" . "Jenney")
      ("Age" . 42)
      ("Revenue" . 180000))
    ("Revenue" . 80000)
    ("Score"
      ("Math" ("School" . 35) ("Exam" . 40))
      ("Literature" ("School" . 50) ("Exam" . 30))))
  (("Number" . 7) ("Name" . "Anne") ("Gender" . "female") ("Age" . 18)
    ("Father"
      ("Number" . 8)
      ("Name" . "Alex")
      ("Age" . 40)
      ("Revenue" . 200000))
    ("Mother"
      ("Number" . 9)
      ("Name" . "Sicie")
      ("Age" . 43)
      ("Revenue" . 50000))
    ("Revenue" . 120000)
    ("Score"
      ("Math" ("School" . 40) ("Exam" . 40))
      ("Literature" ("School" . 30) ("Exam" . 40)))))
```

The `json` library also includes the following procedures for working with JSON data: `json-drop`, `json-push`, and `json-reduce`. See the [documentation](https://guenchi.github.io/json/) for more information on these procedures.

All of the examples have involved reading JSON from strings and displaying JSON in the REPL. The following procedures can be used to read and write JSON files.

```
;; modified from http://rosettacode.org/wiki/Read_entire_file#Scheme
(define (file->string path)
  (with-input-from-file path
    (lambda ()
      (let loop ((char (read-char))
                 (result '()))
        (cond [(eof-object? char)
               (list->string (reverse result))]
              [(member char (list #\newline #\return))
               (loop (read-char) result)]
              [else
               (loop (read-char) (cons char result))])))))

(define (read-json path)
  (string->json (file->string path)))
  
(define (write-json obj path)
  (call-with-output-file path
    (lambda (output-port)
      (display (json->string obj) output-port))))
```
