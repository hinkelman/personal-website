+++
title = "Generating random numbers in R and Racket"
date = 2019-04-05
[taxonomies]
tags = ["R", "Racket"]
+++

R makes it easy to generate random numbers from a wide variety of distributions with a consistent interface. For example, `runif`, `rnorm`, and `rpois` generate random numbers from uniform, normal, and Poisson distributions, respectively. 

<!-- more -->

```
> x = runif(n = 1000, min = 4.6, max = 9.3)
> min(x)
[1] 4.60374
> max(x)
[1] 9.288063
> 
> y = rnorm(n = 1000, mean = 81, sd = 9)
> mean(y)
[1] 81.2548
> sd(y)
[1] 9.179427
> 
> z = rpois(n = 1000, lambda = 7.11)
> mean(z)
[1] 7.187
> var(z)
[1] 6.954986
```

The Racket `math` library provides much of the same functionality as the base R functions, but I first want to work through a couple of examples without using the `math` library because I think it is useful for better understanding Racket.

Let's create a Racket function that is similar to `runif` in R.

```
#lang racket/base

(define (runif n [min 0] [max 1])
  (for/list ([i (in-range n)])
    (+ (* (- max min) (random)) min)))
```

The `random` function draws a random number from a uniform distribution between 0 and 1. A list of `n` random samples is drawn by looping through a sequence with `for/list`. To draw a random number within a specified interval, we multiply the random number by the specified range `(- max min)` and shift by the `min`. 

```
> (define unif (runif 1000 -7 33))
> (apply min unif)  ; check that min is about -7
-6.987705647350004
> (apply max unif)  ; check that max is about 33
32.97250528872971
```

The need to use `apply` to calculate the `min` of a list was a surprise to me. The `min` function takes any number of real numbers as arguments but not a list. `apply` uses the contents of the list as the arguments for `min`.

```
> (min 3 1 2)
1
> (min '(3 1 2))
min: contract violation
  expected: real?
  given: '(3 1 2)
```

Another wrinkle is introduced if you want to generate a vector (with `for/vector`) of random numbers rather than a list.

```
(define (runif-vec n [min 0] [max 1])
  (for/vector ([i (in-range n)])
    (+ (* (- max min) (random)) min)))
```

We can't use `apply` with a vector.

```
> (define unif-vec (runif-vec 1000 0.5 1.5))
> (apply min unif-vec)
apply: contract violation
  expected: list?
  given: '#(1.3465423772765357 1.0623864994795045 1.3437265664094888 1.1048441535810904 0.7861090566755002 0.6470981723620602 0.5546597487687198 0.552946824350613 0.5251621516034304 1.3568612624022975 1.1217164560959263 1.3334770245391927 0.890117942389224...
  argument position: 2nd
  other arguments...:
```

StackOverflow provides a [solution](https://stackoverflow.com/a/52917481/2912447) based on `for/fold`. 

```
> (for/fold ([m +inf.0]) ([x (in-vector unif-vec)]) (min m x))
0.5014616191163698
> (for/fold ([m -inf.0]) ([x (in-vector unif-vec)]) (max m x))
1.4994848021056595
```

In `for/fold`, `m` is an accumulator that is initialized as positive (`+inf.0`) or negative infinity (`-inf.0`) for min and max, respectively. When iterating through `unif-vec`, `m` is compared to `x` and the value of `m` is updated with the output of `min` or `max`.

Random values can also be drawn from a normal distribution using draws from the standard uniform distribution with `random` and a formula from [Rosetta Code](https://rosettacode.org/wiki/Random_numbers#Racket)

```
> (define (rnorm n [mean 0] [sd 1])
   (for/list ([i (in-range n)])
    (+ mean (* (sqrt (* -2 (log (random)))) (cos (* 2 pi (random))) sd))))
> (define norm (rnorm 1000 2 0.5))
> (require math)      ; need math library for mean and standard deviation
> (mean norm)         ; check that mean is about 2
1.9797112583083123
> (stddev norm)       ; check that sd is about 0.5
0.5022756727111091
```

Now, let's simplify our life and use the functions provided by the `math` library.

```
(require math) 

(define (rnorm-2 n [mean 0] [sd 1])
  (if (and (real? mean) (real? sd))
      (flnormal-sample (real->double-flonum mean) (real->double-flonum sd) n)
      (error "mean and sd arguments must be real numbers")))
```

The function, `flnormal-sample`, requires that the mean and standard deviation are passed as floating-point numbers (and returns `flvector`). In `rnorm-2`, we check if the `mean` and `sd` arguments are real numbers and, if so, convert them to floating-point numbers (`real->double-flonum`) before calling `flnormal-sample`, which is otherwise similar to R's `runif` function. 

```
> (define norm-2 (rnorm-2 1000 2 0.5))
> (mean norm-2)
2.024086073688603
> (stddev norm-2)
0.5039448247564015
```

The `math` library contains similar sample functions for numerous types of distributions, including uniform, Poisson, beta, binomial, gamma, and more.

Finally, let's take a look at the `statistics` object provided by the `math` library.

```
> (define s (update-statistics* empty-statistics norm-2))
> (statistics-mean s)
1.9947322187693184
> (statistics-stddev s)
0.4974799838849746
```

`empty-statistics` is an empty statistics object. `update-statistics*` populates the statistics object by iterating through `norm-2` to compute summary statistics, which are extracted with accessor functions, e.g., `statistics-min`, `statistics-mean`, `statistics-skewness`.

The question is why would you use the more verbose statistics object. One reason is that `statistics-min` and `statistics-max` are actually less verbose than using `for/fold` with `min` and `max`. Also, if you are computing several summary statistics on a large sequence, then `update-statistics*` is faster than the other assorted functions.

```
> (define norm-3 (rnorm-2 10000000 1000 100))
> (time
   (mean norm-3)
   (stddev norm-3)
   (variance norm-3)
   (for/fold ([m +inf.0]) ([x (in-flvector norm-3)]) (min m x))
   (for/fold ([m -inf.0]) ([x (in-flvector norm-3)]) (max m x)))
cpu time: 72618 real time: 108130 gc time: 34828
> (time
   (define s2 (update-statistics* empty-statistics norm-3))
   (statistics-mean s2)
   (statistics-stddev s2)
   (statistics-variance s2)
   (statistics-min s2)
   (statistics-max s2))
cpu time: 12478 real time: 22668 gc time: 5512
```
