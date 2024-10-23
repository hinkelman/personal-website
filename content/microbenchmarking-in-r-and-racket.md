+++
title = "A simple microbenchmarking function in Racket"
date = 2019-04-20
[taxonomies]
tags = ["R", "Racket"]
+++

In a [previous post](/post/stochastic-population-model-r-racket/), I wrote a function to perform repeated timings of untyped and typed versions of the same Racket functions. 

<!-- more -->

```
#lang racket

(require math)

(define (time-apply-cpu-old proc lst reps)
  (define out
    (for/list ([i (in-range reps)])
      (define-values (results cpu-time real-time gc-time) (time-apply proc lst))
      cpu-time))
  (displayln (string-append "min: " (number->string (apply min out))
                            " mean: " (number->string (round (mean out)))
                            " max: " (number->string (apply max out))
                            "    function: "
                            (symbol->string (object-name proc)))))
```

`time-apply-cpu-old` is a wrapper function for `time-apply` from `racket/base` that runs `time-apply` repeatedly and prints the `min`, `mean`, and `max` cpu time. `time-apply` produces multiple output values that are not contained in a data structure. `define-values` allows you to capture those outputs and bind them to names (in this case, `results`, `cpu-time`, `real-time`, `gc-time`). `string-append` is similar to `paste` in R, but `string-append` requires that all arguments are strings and, thus, requires some conversion (e.g., `number->string`).

```
> (time-apply-cpu-old flnormal-sample (list 0.0 1.0 10000) 50)
min: 1 mean: 2 max: 22    function: flnormal-sample
> (time-apply-cpu-old flnormal-sample (list 0.0 1.0 100000) 50)
min: 14 mean: 17 max: 45    function: flnormal-sample
```

`time-apply-cpu-old` only allows for evaluation of a single function at a time and the display of the output is ugly. I thought it would be a good exercise to try to address those two deficiencies. There was a [recent discussion on the Racket mailing list](https://groups.google.com/forum/#!topic/racket-users/7MCIp7RmTh8) that provided several options to allow for pretty printing of the timing output. I opted for a suggestion involving the `table` function from the [raart package](https://docs.racket-lang.org/raart/index.html?q=raart).

```
> (require raart/draw)
> (define example-list (list (list "col1" "col2" "col3")
                             (list 1.001 2.002 3.003)
                             (list 4.004 5.005 6.006)))
> (draw-here
   (table
    (text-rows example-list)
    #:frames? #f
    #:inset-dw 1
    #:halign 'right))
    
  col1   col2   col3 
 1.001  2.002  3.003 
 4.004  5.005  6.006 
```

In this example, I only changed a few of the default arguments to the `table` function. I dropped the table borders, increased the horizontal spacing from 0 to 1, and changed the horizontal alignmental from `'left` to `'right`. 

I'm modeling the format for my target output on the `microbenchmark` function in the [microbenchmark package](https://cran.r-project.org/web/packages/microbenchmark/index.html) for R. 

```
> library(microbenchmark)
> microbenchmark(rnorm(10000), rnorm(100000), unit = "ms")
Unit: milliseconds
         expr      min       lq      mean    median        uq       max neval
 rnorm(10000) 0.575444 0.618125 0.7051323 0.6259115 0.6549435  4.466793   100
 rnorm(1e+05) 5.741617 6.131754 6.3758384 6.1757695 6.4165000 11.183758   100
```

First, we are going to modify `time-apply-cpu-old` to return a list of `cpu-time` rather than displaying the `min`, `mean`, and `max` cpu time.

```
(define (time-apply-cpu proc args reps)
  (for/list ([i (in-range reps)])
    (define-values (results cpu-time real-time gc-time) (time-apply proc args))
    cpu-time))
```

My new Racket function, `microbenchmark`, requires similar arguments as `time-apply-cpu`, but `procs` is a list, `args` is a list of lists, and reps has a default value of 100. `microbenchmark` starts with an expression to check that the lengths of `procs` and `args` match. `(unless (equal? (length procs) (length args))` is similar to `if (length(procs) != length(args))` in R. I don't yet have a good sense for how much effort I should put into checking inputs, but my preliminary impression is that Racket is less likely than R to run successfully with unexpected inputs (and with potentially incorrect outputs).

```
(define (microbenchmark procs args [reps 100])
  (unless (equal? (length procs) (length args))
    (error "List of procedures is not same length as list of arguments."))
  (define (create-timing-table procs args [result (list (list "expr" "args" "min" "lq" "mean" "median" "uq" "max" "neval"))])
    (cond
      [(null? procs) (reverse result)]
      [else
       (define tmp (time-apply-cpu (first procs) (first args) reps))
       (create-timing-table
        (rest procs)
        (rest args)
        (cons (list (symbol->string (object-name (first procs)))
                    (first args)
                    (apply min tmp)
                    (quantile 0.25 < tmp)
                    (round (mean tmp))
                    (quantile 0.5 < tmp)
                    (quantile 0.75 < tmp)
                    (apply max tmp)
                    reps)
              result))]))
  (displayln "Units: milliseconds")
  (draw-here (table
              (text-rows (create-timing-table procs args))
              #:frames? #f
              #:inset-dw 1
              #:halign 'right)))
```

Within the `microbenchmark` function, a recursive function, `create-timing-table`, repeatedly calls `time-apply-cpu` to build up a table of results. The results table is initialized as a list of lists where the first list contains the column headings. `create-timing-table` is a list-eater function that passes the first item from `procs` and `args` to `time-apply-cpu` and then uses `cons` to add the latest output to the front of the results list, which is why the results list needs to be reversed when the function exits. 

To demonstrate the `microbenchmark` output, I will compare two different functions for drawing random numbers from a normal distribution based on [this post](/post/generating-random-numbers-r-racket/).

```
> (microbenchmark (list flnormal-sample
                        rnorm
                        flnormal-sample
                        rnorm)
                  (list (list 0.0 1.0 10000)
                        (list 0.0 1.0 10000)
                        (list 0.0 1.0 100000)
                        (list 0.0 1.0 100000)))
Units: milliseconds
            expr              args  min  lq  mean  median  uq   max  neval 
 flnormal-sample   (0.0 1.0 10000)    1   1     2       1   2     8    100 
           rnorm   (0.0 1.0 10000)    4   5     7       5   9    34    100 
 flnormal-sample  (0.0 1.0 100000)   14  15    15      15  16    21    100 
           rnorm  (0.0 1.0 100000)   62  67    93      73  84  1772    100 
```
