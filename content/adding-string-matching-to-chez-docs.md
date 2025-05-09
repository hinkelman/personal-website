+++
title = "Adding string matching to chez-docs"
date = 2020-01-05
updated = 2025-05-08
[taxonomies]
tags = ["chez-docs", "Chez Scheme"]
+++

I recently wrote a little library, [`chez-docs`](https://github.com/hinkelman/chez-docs), to make accessing documentation easier while learning Chez Scheme ([blog post](/post/access-chez-scheme-documentation-from-repl/)). The main procedure, `doc`, in `chez-docs` only returns results for exact matches with `proc` [[1]](#1). To aid in discovery, I've added a procedure, `find-proc`, that provides exact and approximate matching of search strings.

<!-- more -->

### Levenshtein Distance

My initial thought was that I should approach this problem with approximate string matching. After a little searching, I learned that [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance) was one of the simplest approaches to calculate the distance between two strings. This excellent [blog post](https://blogs.mathworks.com/cleve/2017/08/14/levenshtein-edit-distance-between-strings/) included a few MATLAB implementations of Levenshtein distance algorithms [[2]](#2) that were relatively easy for me to follow because of my experience with MATLAB and R. 

I first implemented the recursive algorithm [[3]](#3) thinking that it would be most natural in Scheme, but it was unacceptably slow. I then implemented the iterative two-row algorithm and found the performance to be sufficiently snappy for my needs.

```
(define (lev s t)
  (let* ([s (list->vector (string->list s))]
         [t (list->vector (string->list t))]
         [m (vector-length s)]
         [n (vector-length t)]
         [x (list->vector (iota (add1 n)))]
         [y (list->vector (make-list (add1 n) 0))])
    (do ((i 0 (add1 i)))
        ((= i m))
      (vector-set! y 0 i)
      (do ((j 0 (add1 j)))
          ((= j n))
        (let ([c (if (char=? (vector-ref s i) (vector-ref t j)) 0 1)])
          (vector-set! y (add1 j) (min (add1 (vector-ref y j))
                                       (add1 (vector-ref x (add1 j)))
                                       (+ c  (vector-ref x j))))))
      ;; swap x and y
      (let ([tmp x])
        (set! x y)
        (set! y tmp)))
    (vector-ref x n)))
```

This is the first time that I've used `do` loops in Scheme. In the example below, the looping index `i` is initialized to zero and incremented by one on each pass through the loop. The loop is exited when `(= i 10)`. The equivalent code in R is `for (i in 0:9) cat(paste0(i, " "))`.

```
> (do ((i 0 (add1 i)))
      ((= i 10))
    (display (string-append (number->string i) " ")))
0 1 2 3 4 5 6 7 8 9
```

`lev` tallies the numbers of insertions, deletions, and substitutions; a value of zero indicates an exact match. 

```
> (map (lambda (x) (lev "head" x)) '("head" "read" "load" "list-head"))
(0 1 2 5)
```

### Exact Substring Matching

`doc` uses [`assoc`](https://scheme.com/tspl4/objects.html#./objects:s58) to find any exact matches of the full string in the list of procedures. After working with the Levenshtein distance, I realized that exact matching of substrings would generally be more useful than fuzzy matching. I wrote the `string-match` procedure to test if a search string is present in the target string.

```
(define (string-match s t)
  (let* ([s-list (string->list s)]
         [t-list (string->list t)])
    (if (char=? (car s-list) #\^)
        (string-match-helper (cdr s-list) t-list)
        (not (for-all (lambda (x) (equal? x #f))
                      (map (lambda (t-sub) (string-match-helper s-list t-sub))
                           (potential-matches (car s-list) t-list)))))))

;; loop through characters in search string
;; to check if search string is found in target string
(define (string-match-helper s-list t-list)
  (cond [(not t-list) #f] 
        [(null? s-list) #t]
        [(< (length t-list) (length s-list)) #f]
        [(char=? (car s-list) (car t-list))
         (string-match-helper (cdr s-list) (cdr t-list))]
        [else #f]))

;; loop through target string
;; to find all potential substring matches
(define (potential-matches char t-list)
  (let loop ([t-list t-list]
             [results '()])
    (if (null? t-list)
        (remove-duplicates (reverse results))
        (loop (cdr t-list) (cons (member char t-list) results)))))
  
(define (remove-duplicates ls)
  (cond [(null? ls)
         '()]
        [(member (car ls) (cdr ls))
         (remove-duplicates (cdr ls))]
        [else
         (cons (car ls) (remove-duplicates (cdr ls)))]))
```

`member` is the workhorse of `string-match` (via `potential-matches`). It's an interesting turn for me because when I first started using `member` in my Scheme code I was puzzled by why it didn't work like `%in%` in R. For example, `(member 2 '(1 2 3))` returns `(2 3)`, but `2 %in% c(1, 2, 3)` returns `TRUE`. Because all values other than `#f` count as `#t` in Scheme, `member` can be used as a predicate, e.g., `(if (member 2 '(1 2 3)) 1 0)` returns `1`. Nonetheless, it wasn't obvious to me how `member`'s behavior was useful...until I started writing `string-match`. Those experiences make programming fun.

`string-match` returns a boolean value.

```
> (map (lambda (x) (string-match "head" x)) '("head" "read" "load" "list-head"))
(#t #f #f #t)
```

### Procedure Discovery

`find-proc` takes a `search-string` and two optional arguments, `search-type` and `max-results`, which default to `'exact` and `10`, respectively.

```
(define find-proc
  (case-lambda
    [(search-string)
      (find-proc-helper search-string 'exact 10)]
    [(search-string search-type)
      (find-proc-helper search-string search-type 10)]
    [(search-string search-type max-results)
      (find-proc-helper search-string search-type max-results)]))
```

`find-proc-helper` maps either `lev` or `string-match` to the full list of procedures, `proc-list`, and then sorts or filters the results, respectively.

```
(define (find-proc-helper search-string search-type max-results)
  (unless (string? search-string)
    (assertion-violation "(find-proc search-string)" "search-string is not a string"))
  (cond [(symbol=? search-type 'fuzzy)
        (let* ([dist-list (map (lambda (x) (lev search-string x))
                                proc-list)]
                [dist-proc (map (lambda (dist proc) (cons dist proc))
                                dist-list proc-list)]
                [dist-proc-sort (sort (lambda (x y) (< (car x) (car y)))
                                      dist-proc)])
          (prepare-results dist-proc-sort search-type max-results))]
        [(symbol=? search-type 'exact)
        (let* ([bool-list (map (lambda (x) (string-match search-string x))
                                proc-list)]
                [bool-proc (map (lambda (bool proc) (cons bool proc))
                                bool-list proc-list)]
                [bool-proc-filter (filter (lambda (x) (car x)) bool-proc)])
          (prepare-results bool-proc-filter search-type max-results))]
        [else
        (assertion-violation "(find-proc search-string search-type)"
                              "search-type must be either 'exact or 'fuzzy")]))

(define (prepare-results ls search-type max-results)
  (let* ([len (length ls)]
          [max-n (if (> max-results len) len max-results)])
    (when (and (symbol=? search-type 'exact) (> len max-results))
      (display (string-append "Returning " (number->string max-results)
                              " of " (number->string len)
                              " results\n")))
    (map cdr (list-head ls max-n))))
```

I first realized that Levenshtein distance might not be very useful for `find-proc` when searching for `head`, a commonly used procedure in R.

```
> (find-proc "head" 'fuzzy 5)
("read" "and" "cadr" "car" "cd")
```

However, substring matching points us to the relevant function, `list-head`, in Chez Scheme. 

```
> (find-proc "head" 'exact 5)
("list-head" "lookahead-char" "lookahead-u8" "make-boot-header")
```

Fuzzy matching is useful, though, for discovery when there are options with similar forms, e.g., `hash-table?` and `hashtable?`.

```
> (find-proc "hash-table?" 'exact 3)
("hash-table?")
> (find-proc "hash-table?" 'fuzzy 3)
("hash-table?" "hashtable?" "eq-hashtable?")
```

The `^` indicates that only search strings found at the start of the procedure should be returned.

```
> (find-proc "map")
("andmap" "hash-table-map" "map" "ormap" "vector-map")
> (find-proc "^map")
("map")

> (find-proc "file" 'exact 3)
Returning 3 of 78 results
("&i/o-file-already-exists"
  "&i/o-file-does-not-exist"
  "&i/o-file-is-read-only")
  
> (find-proc "^file" 'exact 3)
("file-access-time" "file-buffer-size" "file-change-time")

> (find-proc "let" 'exact 5)
Returning 5 of 20 results
("delete-directory"
  "delete-file"
  "eq-hashtable-delete!"
  "fluid-let"
  "fluid-let-syntax")

> (find-proc "^let" 'exact)
("let" "let*" "let*-values" "let-syntax" "let-values"
  "letrec" "letrec*" "letrec-syntax")
```

Under fuzzy matching, the `^` is included as part of the Levenshtein distance calculation and, thus, should not be included in search strings when using fuzzy matching.

```
> (find-proc "map" 'fuzzy 5)
("map" "max" "*" "+" "-")
> (find-proc "^map" 'fuzzy 5)
("map" "max" "car" "exp" "memp")
```

UPDATE (2025-05-08): I realized recently that one of my favorite little elements of `chez-docs` is that, if `doc` can't find match, it returns a guess powered by `find-proc` and fuzzy matching. 

```
> (doc "ifelse")
Exception in (doc proc): ifelse not found in csug or tspl
Did you mean 'else'?
```

Searching for `ifelse` (from R) and turning up `else` isn't the most direct route to understanding, but it will lead you to `cond` and get you to a little closer to where you want to go.


It took me about three years to realize that it would be an easy addition (basically 3 lines of code added in July 2023) and another two years to realize that I should mention it somewhere.

```
(define (guess-proc proc)
  (string-append "Did you mean '" (car (find-proc proc 'fuzzy)) "'?"))
```

It has the same limitations of fuzzy matching as described above, but it is helpful if you simply introduced a typo or a misplaced dash in your call to `doc`. 

```
> (doc "make-byte-vector")
Exception in (doc proc): make-byte-vector not found in csug or tspl
Did you mean 'make-bytevector'?
```

***

<a name="1"></a> [1] `proc` is shorthand for procedure, but not all of the items in `chez-docs` are procedures, e.g., `&assertion`.

<a name="2"></a> [2] The MATLAB post linked to implementations of Levenshtein distance in other languages, including [Scheme](https://en.wikibooks.org/wiki/Algorithm_Implementation/Strings/Levenshtein_distance#Scheme), but the Scheme example was hard for me to follow so I set it aside.

<a name="3"></a> [3] After translating the MATLAB version of the recursive algorithm to Chez Scheme, I realized that a recursive example was available for Scheme on [Rosetta Code](http://rosettacode.org/wiki/Levenshtein_distance#Scheme).