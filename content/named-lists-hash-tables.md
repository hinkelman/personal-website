+++
title = "Storing parameters in named lists and hash tables in R and Racket"
date = 2019-03-20
[taxonomies]
categories = ["R", "Racket"]
tags = ["named-lists", "hashtables", "data-structures"]
+++

When building a simulation model in R, I might want to group related input parameters into a data structure. For example, in a life cycle model with resident and anadromous fish, you might use different fecundity parameters for each life history type. One option is to create different objects for each fecundity parameter.

<!-- more -->

```
fecundity_resident = 1000
fecundity_anadromous = 4000
```

That option is not unreasonable with only two fecundity parameters but it becomes cluttered with many values. You can combine these values into a named vector.

```
# create named vector
fecundity = c("resident" = 1000,
              "anadromous" = 4000)
```

The fecundity value is then looked up by name, which is safer than by position, e.g., `fecundity[["resident"]]` returns `1000` [[1]](#1).

When a parameter differs across multiple groups, a nested named list fits the bill. In the next example, the spawning probability parameter is different between sexes and across age classes. The nested list structure is easy to read and indexing is similar to using a named vector.

```
# create nested named list
spawn_prob = list("female" = list("0-1" = 0, 
                                  "1-2" = 0,
                                  "2-3" = 0.15,
                                  "3-4" = 0.4,
                                  "4+" = 0.8),
                  "male" = list("0-1" = 0, 
                                "1-2" = 0.1,
                                "2-3" = 0.4,
                                "3-4" = 0.6,
                                "4+" = 0.9))
# index nested named list
spawn_prob[["male"]][["3-4"]]
```

I am currently [learning Racket](/post/programming-horizons/) by trying to [translate small examples of R code](/post/for-loop-r-racket) into Racket code. A big part of that translation is understanding the differences between the data structures in [R](http://adv-r.had.co.nz/Data-structures.html) and [Racket](https://beautifulracket.com/explainer/data-structures.html). 

Racket has a [list data structure](https://docs.racket-lang.org/reference/pairs.html) but values can only be referenced by position, not name. However, Racket's hash tables provide a similar structure to R's named lists, which allows for an easy translation. 

```
; create nested hash table
(define spawn-prob
  (hash "female" (hash "0-1" 0
                       "1-2" 0
                       "2-3" 0.15
                       "3-4" 0.4
                       "4+" 0.8)
        "male" (hash "0-1" 0
                     "1-2" 0.1
                     "2-3" 0.4
                     "3-4" 0.6
                     "4+" 0.9)))
; index nested hash table
(hash-ref (hash-ref spawn-prob "male") "3-4")
```

Update: After a little more exposure to hash tables in Racket, it seems that it is more idiomatic to represent keys as symbols than strings. At the very least, it saves some keystrokes. 

```
(define spawn-prob
  (hash 'female (hash '0-1 0
                      '1-2 0
                      '2-3 0.15
                      '3-4 0.4
                      '4+ 0.8)
        'male (hash '0-1 0
                    '1-2 0.1
                    '2-3 0.4
                    '3-4 0.6
                    '4+ 0.9)))

(hash-ref (hash-ref spawn-prob 'male) '3-4)
```

***

<a name="1"></a> [1] Note that indexing with `[` returns the name and the value. Using `[[` returns the value only.