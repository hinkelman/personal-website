+++
title = "Spam simulation in Chez Scheme"
date = 2020-12-12
[taxonomies]
categories = ["Chez Scheme", "dataframe", "chez-stats"]
tags = ["simulation", "random-variates"]
+++

I learned a lot about Scheme by writing a few Chez Scheme libraries and I expect that there is more to learn by trying to use those libraries (e.g., [EDA in Chez Scheme](/eda-chez-scheme/)). A [blog post](http://varianceexplained.org/r/spam-simulation/) about a stochastic simulation of spam comments in R caught my eye as an interesting example to test my [`chez-stats`](https://github.com/hinkelman/chez-stats) and [`dataframe`](https://github.com/hinkelman/dataframe/) libraries.

<!-- more -->

I was drawn to the spam simulation post because it describes using the `crossing` function from the [`tidyr` package](https://tidyr.tidyverse.org/) as a convenient way to examine the parameter space of a simulation under the familar framework of a dataframe, which is an approach that I've used in my own work. One of the challenges for me in learning new programming languages is that my R experience has led me to primarily think in terms of dataframes. 

The takeway from implementing the spam simulation in Chez Scheme is that my `dataframe` library is not well suited to this task. That is not entirely surprising. `dataframe` was written for ease of implementation, not performance. More importantly, for a simple simulation like this, plain Scheme code provides a more straightforward solution than the `dataframe` approach [[1]](#1). 

In this post, I will focus on explaining the Scheme code rather than describing the simulation. In short, the idea is that you can build up your intuition about a problem with simulations, which might eventually lead you to a more concise mathematical solution to the problem. The objective is to determine the number of spam comments after three days given that spammers follow a Poisson process. 

### Dataframe approach

First, let's import the necessary libraries (after following installation instructions in the repos linked above).

```
(import (chez-stats)
        (dataframe))
```

`dataframe-crossing` takes an `alist` and returns the cartesian product of the values in the alist. The original post used 25,000 trials. We are only using 50 trials because the performance of this solution is poor. We are using the `->` operator to build up a chain of operations in a more readable way.

```
(define sim-waiting
  (-> (list (cons 'trial (map add1 (iota 50)))
            (cons 'time (map (lambda (x) (* x 0.25)) (iota 13)))
            (cons 'observation (map add1 (iota 300))))
      (dataframe-crossing)))

> (dataframe-dim sim-waiting)
(195000 . 3)

> (dataframe-display sim-waiting)
     trial      time  observation
         1         0            1
         1         0            2
         1         0            3
         1         0            4
         1         0            5
         1         0            6
         1         0            7
         1         0            8
         1         0            9
         1         0           10
```

We continue to build up the `sim-waiting` dataframe by adding a column with waiting times by drawing from an exponential distribution with the observation as the rate parameter (used by `rexp` in R). However, `random-exponential` from `chez-stats` takes the mean of the distribution as the parameter, which is equal to `1/rate`.  

```
> (define sim-waiting
   (-> (list (cons 'trial (map add1 (iota 0)))
             (cons 'time (map (lambda (x) (* x 0.25)) (iota 13)))
             (cons 'observation (map add1 (iota 300))))
       (dataframe-crossing)
       (dataframe-modify
        (modify-expr (waiting (observation)
                              (random-exponential (/ 1 observation)))))))

> (dataframe-display sim-waiting)
     trial      time  observation   waiting
         1         0            1  0.524573
         1         0            2  0.143351
         1         0            3  0.372225
         1         0            4  0.012361
         1         0            5  0.093031
         1         0            6  0.055577
         1         0            7  0.663598
         1         0            8  0.147212
         1         0            9  0.123309
         1         0           10  0.145412
```

The next step uses the `split-apply-combine` strategy to return the cumulative sum of the `waiting` column for each `trial`/`time` combination. `->` pipes into the first argument of the next procedure whereas `->>` pipes into the last. In the apply step, we add a new column for each dataframe that came out of `dataframe-split` with `(cumulative () (cumulative-sum ($ df 'waiting)))`. If a `modify-expr` contains a list of the same length as the number of rows in the dataframe, then `dataframe-modify` adds that list to the dataframe as a column with the specified name, which is `cumulative` in this example.

```
> (define sim-waiting
   (-> (list (cons 'trial (map add1 (iota 50)))
             (cons 'time (map (lambda (x) (* x 0.25)) (iota 13)))
             (cons 'observation (map add1 (iota 300))))
       (dataframe-crossing)
       (dataframe-modify
        (modify-expr (waiting (observation)
                              (random-exponential (/ 1 observation)))))
       (dataframe-split 'trial 'time)
       (->> (map (lambda (df)
	           (dataframe-modify
	            df
	            (modify-expr
		     (cumulative () (cumulative-sum ($ df 'waiting))))))))
       (->> (apply dataframe-bind))))

> (dataframe-display sim-waiting)
     trial      time  observation   waiting  cumulative
         1         0            1  0.524573  0.52457379
         1         0            2  0.143351  0.66792542
         1         0            3  0.372225  1.04015060
         1         0            4  0.012361  1.05251217
         1         0            5  0.093031  1.14554345
         1         0            6  0.055577  1.20112111
         1         0            7  0.663598  1.86471927
         1         0            8  0.147212  2.01193168
         1         0            9  0.123309  2.13524144
         1         0           10  0.145412  2.28065430
```

We aggregate `sim-waiting` to find the number of spam comments within each `trial` and `time`. The `cumulative` column gives the total time that has elapsed. We are counting the number of rows where `cumulative` is less than `time` to determine the number of comments received in a specified `time`. 

```
> (define sim-waiting-times
   (-> sim-waiting
       (dataframe-aggregate
        '(trial time)
        (aggregate-expr
         (num-comments (cumulative time)
                       (length (filter (lambda (x) (< x 0))
                                       (map (lambda (cs t) (- cs t))
                                            cumulative time))))))))

> (dataframe-display sim-waiting-times)
     trial      time  num-comments
         1         0             0
         1      0.25             2
         1       0.5             0
         1      0.75             0
         1       1.0             0
         1      1.25             1
         1       1.5             2
         1      1.75             2
         1       2.0            15
         1      2.25             1
```

The last step is to calculate the average number of spam comments for each time across all trials.

```
> (define average-over-time
   (dataframe-aggregate
    sim-waiting-times
    '(time)
    (aggregate-expr
     (average (num-comments) (exact->inexact (mean num-comments))))))

> (dataframe-display average-over-time 13)
      time   average
         0       0.0
      0.25      0.22
       0.5      0.58
      0.75      0.84
       1.0      2.08
      1.25      1.96
       1.5      3.44
      1.75      4.72
       2.0      5.62
      2.25      9.98
       2.5     11.32
      2.75     12.84
       3.0     15.76

```

### Idiomatic Scheme approach

As I mentioned above, the `dataframe` approach is inefficient and more verbose than an idiomatic Scheme approach. One source of inefficiency [[2]](#2) is generating 300 observations per trial when the average for three days is only 19. This inefficiency is not expensive in R because it is well optimized for these types of vectorized operations. However, in Scheme, we can write a simple recursive function that doesn't bother to build up a list of all of the waiting times and stops as soon as the number of comments exceeding the waiting time is known.

```
> (define (get-num-events max-obs max-time)
   (let loop ([total 0]
              [count 0]
              [obs 1])
     ;; arguably silly to have two counting variables; but makes code clearer
     (if (or (> obs max-obs) (> total max-time))
         (sub1 count) ;; sub1 to find count that is less than max-time threshold
         (let ([exp-draw (random-exponential (/ 1 obs))])
           (loop (+ total exp-draw) (add1 count) (add1 obs))))))

> (get-num-events 300 3)
16
> (get-num-events 300 3)
1
> (get-num-events 300 3)
24
> (get-num-events 300 3)
28
```

The `repeat` procedure from `chez-stats` uses recursion to repeat a thunk `n` times, which allows us to build up a set of replications. 

```
> (define sim (repeat 1e5 (lambda () (get-num-events 300 3))))
> (exact->inexact (mean sim))
19.03991
```

The last step is to vary the times used in the simulation.

```
> (define max-times (map (lambda (x) (* x 0.25)) (iota 13)))

> (define sim-times
   (map (lambda (mt)
          (repeat 1e5 (lambda () (get-num-events 300 mt))))
        max-times))

> (define sim-times-mean (map (lambda (x) (exact->inexact (mean x))) sim-times))

> max-times
(0 0.25 0.5 0.75 1.0 1.25 1.5 1.75 2.0 2.25 2.5 2.75 3.0)
> sim-times-mean
(0.0 0.28459 0.64931 1.11663 1.71885 2.48855 3.47136 4.74836
     6.38984 8.48071 11.13153 14.64021 19.07148)
```

This approach uses 100,000 trials and is effectively instantaneous whereas only 50 trials in the `dataframe` approach took a few seconds for every operation and hundreds of trials was likely to freeze Emacs. The `dataframe` library is not capable of working with hundreds of thousands of row. Fortunately, many data analysis projects involve datasets much smaller than that. More importantly, idiomatic Scheme code is well suited for simulations like this spam comments problem.

***

<a name="1"></a> [1] I expect that the `dataframe` library would prove more useful in a data analysis task.

<a name="2"></a> [2] The bigger inefficiency is the `dataframe` library, particularly `dataframe-split`. 