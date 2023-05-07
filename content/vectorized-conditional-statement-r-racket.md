+++
title = "Vectorized conditional statement in R and Racket"
date = 2019-04-15
[taxonomies]
categories = ["R", "Racket"]
tags = ["conditional", "if", "ifelse", "map", "sapply"]
+++

Racket's `if` is not vectorized like `ifelse` in R. Instead, this Racket code

<!-- more -->

```
(if (test-expr)
    true-expr
    false-expr)
```

is the same as this R code.

```
if (test_expr){
  true_expr
} else {
  false_expr
}
```

In contrast, R's `ifelse` function is vectorized meaning that the same operation is applied to multiple elements of a vector. Below, we use `ifelse` to return a vector of the same length as the original vector with all negative values replaced by zero.

```
> a = c(-999, 2, -999, 4, 5, 6, 7, -999, 9, 10)
> ifelse(a < 0, 0, a)
 [1]  0  2  0  4  5  6  7  0  9 10
```

In Racket, we use `map` and an [anonymous function](https://en.wikipedia.org/wiki/Anonymous_function), specified with `lambda`, to apply `if` to the elements of a list.

```
> (define a '(-999 2 -999 4 5 6 7 -999 9 10))
> (map (lambda (x) (if (< x 0) 0 x)) a)
'(0 2 0 4 5 6 7 0 9 10)
```

To apply `if` to a vector, we use `vector-map`.

```
> (define b #(-999 2 -999 4 5 6 7 -999 9 10))
> (vector-map (lambda (x) (if (< x 0) 0 x)) b)
'#(0 2 0 4 5 6 7 0 9 10)
```

It is also possible to write similar code in R using `sapply`. The curly braces can be omitted when `if else` is written inline.

```
> sapply(a, function(x) if (x < 0) 0 else x)
 [1]  0  2  0  4  5  6  7  0  9 10
```

R and Racket have a shared heritage in Scheme and Lisp that can yield some strikingly similar code.
