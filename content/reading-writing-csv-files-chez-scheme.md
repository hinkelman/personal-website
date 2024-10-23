+++
title = "Reading and writing CSV files in Chez Scheme"
date = 2019-10-18
[taxonomies]
tags = ["chez-stats", "Chez Scheme"]
+++

I have added functionality for reading and writing CVS files to my Chez Scheme library, [`chez-stats`](https://github.com/hinkelman/chez-stats). In a [previous post](/post/reading-csv-files-in-r-and-racket/), I compared reading CSV files in R and Racket and made the following observation.

<!-- more -->

> By and large, R users are not programmers but end users who want to expeditiously perform tasks related to cleaning, analyzing, and visualizing their data. There is a large, and growing, industry around building R packages and tools that facilitate those end users. My early impression of the Racket community is that packages are generally written at a lower level of abstraction than R packages because the target audience is other programmers.

Well, writing procedures for reading and writing CSV files in Chez Scheme required me to take yet another step down the abstraction ladder. As an R programmer, I have given very little thought to the mechanics of reading and writing files. When I started working with examples of parsing files into Scheme data structures, I was honestly suprised to realize that the contents of a file are parsed character by character. Discovering these gaping holes in my understanding of programming and computer science is alarming. But that is largely the point of me spending my free time learning Scheme. 

### Reading a CSV file

[Rosetta Code](http://rosettacode.org/wiki/Category:Programming_Tasks) was my first stop to learn more about how to approach reading files. The two approaches provided there are to read a file line by line or to read the whole file into a string. I opted for reading line by line and the Scheme example for that task involved a function called `read-line`. In Chez Scheme, though, the function is called `get-line`, not `read-line`. Unfortunately, `get-line` only breaks lines for the line feed character (`\n`), not the carriage return character (`\r`).

```
> (get-line (open-input-string "One,2,C,867-5309\r\nTwo,3,D,555-2439\r\n"))
"One,2,C,867-5309\r"
```

That problem was easily solved thanks to a helpful [StackOverflow](https://stackoverflow.com/questions/37858083/how-to-read-a-line-of-input-in-chez-scheme) user from 2016 who provided a `read-line` procedure.

```
> (read-line (open-input-string "One,2,C,867-5309\r\nTwo,3,D,555-2439\r\n"))
"One,2,C,867-5309"
```

Now that we can read a file line by line, we need a procedure to parse each line (represented as a string) into a list. For parsing, I'm using a [`parse-line` procedure](https://github.com/alex-hhh/data-frame/blob/master/private/csv.rkt) written by [Alex Harsanyi](https://alex-hhh.github.io/About.html) for Racket as part of his [data-frame package](https://docs.racket-lang.org/data-frame/index.html). 

```
> (parse-line (read-line (open-input-string "One,2,C,867-5309\r\nTwo,3,D,555-2439\r\n")))
("One" "2" "C" "867-5309")
```

Both `read-line` and `parse-line` read every character. I pondered trying to combine the two procedures to remedy that inefficiency. Ultimately, though, I decided it was better to have two more easily understood procedures and live with the inefficiency. That is consistent with the rest of `chez-stats`, which is **not** written with performance in mind.

With `read-line` and `parse-line` in our toolboox, we can iterate through the CSV file to build up a list of lists where each sub-list is one row in the CSV file. There is nothing sophisticated about my approach to reading CSV files. For all files, we end up with a list of lists of strings. There is no attempt to convert strings to numbers or other objects. Also, the CSV file needs to be rectangular, i.e., every row must have the same number of columns. 

### Writing a CSV file

We can write a file line by line with the `put-string` procedure provided by Chez Scheme. First, we need to write a procedure to handle strings that contain commas and double quotes.

```
(define (quote-string str sep-char)
  (let* ([in (open-input-string str)]
         [str-list (string->list str)]
         [str-length (length str-list)])
    (if (not (or (member sep-char str-list) (member #\" str-list)))
        str  ;; return string unchanged b/c no commas or double quotes
        (let loop ([c (read-char in)]
                   [result "\""]
                   [ctr 0])
          (cond [(eof-object? c)
                 (string-append result "\"")]
                [(and (char=? c #\") (or (= ctr 0) (= ctr (sub1 str-length))))
                 ;; don't add double-quote character to string
                 ;; when it is at start or end of string
                 (loop (read-char in) (string-append result "") (add1 ctr))]
                ;; 2x double-quotes for double-quotes inside string (not at start or end)
                [(char=? c #\")
                 (loop (read-char in) (string-append result "\"\"") (add1 ctr))]
                [else
                 (loop (read-char in) (string-append result (string c)) (add1 ctr))])))))
```

`quote-string` returns simple strings unchanged.

```
> (parse-line "example" #\,)
("example")
> (parse-line (quote-string "example" #\,) #\,)
("example")
```

Strings that contain commas and double quotes are double quoted.

```
> (parse-line "1,000" #\,)
("1" "000")
> (parse-line (quote-string "1,000" #\,) #\,)
("\"1,000\"")
> (parse-line "Earvin \"Magic\" Johnson" #\,)
("Earvin \"Magic\" Johnson")
> (parse-line (quote-string "Earvin \"Magic\" Johnson" #\,) #\,)
("\"Earvin \"Magic\" Johnson\"")
```

But strings containing single quotes are not modified.

```
> (parse-line "Earvin 'Magic' Johnson" #\,)
("Earvin 'Magic' Johnson")
> (parse-line (quote-string "Earvin 'Magic' Johnson" #\,) #\,)
("Earvin 'Magic' Johnson")
```

The `delimit-list` procedure converts the elements of the list from characters, symbols, and numbers to strings before appending the elements into a single string separated by commas. Moreover, exact numbers are converted to inexact numbers before converting to strings.

```
(define (delimit-list ls sep-char)
  (let loop ([ls ls]
             [result ""]
             [first? #t])
    (if (null? ls)
        result
        (let* ([item (car ls)]
               [sep-str (if first? "" (string sep-char))]
               [item-new (cond [(char? item) (string item)]
                               [(symbol? item) (symbol->string item)]
                               [(real? item) (number->string
                                              (if (exact? item)
                                                  (exact->inexact item)
                                                  item))]
                               [else (quote-string item sep-char)])])
          (loop (cdr ls) (string-append result sep-str item-new) #f)))))
```

```
> (delimit-list (list 'a #\b "c" 7 1/3) #\,)
"a,b,c,7.0,0.3333333333333333"
```

In `write-delim`, we loop through a list of lists, convert each row to a string, and write that string to a file at the specified path. 

```
(define write-delim
  (case-lambda
    [(ls path) (write-delim-helper ls path #\, #t)]
    [(ls path sep-char) (write-delim-helper ls path sep-char #t)]
    [(ls path sep-char overwrite) (write-delim-helper ls path sep-char overwrite)]))

(define (write-delim-helper ls path sep-char overwrite)
  (when (and (file-exists? path) (not overwrite))
    (assertion-violation path "file already exists"))
  (delete-file path)
  (let ([p (open-output-file path)])
    (let loop ([ls-local ls])
      (cond [(null? ls-local)
             (close-port p)]
            [else
             (put-string p (delimit-list (car ls-local) sep-char))
             (newline p)
             (loop (cdr ls-local))]))))
```


