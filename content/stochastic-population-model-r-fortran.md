+++
title = "Stochastic population model in R, Rcpp, and Fortran"
date = 2021-05-09
[taxonomies]
categories = ["Fortran", "R"]
tags = ["stochastic-population-model", "Rcpp", "timing"]
+++

The stochastic logistic population model described in [this blog post](https://www.seascapemodels.org/rstats/2017/02/26/speeding-up-sims.html) has become my default exercise when I'm exploring a new programming language ([Racket](/stochastic-population-model-r-racket/), [F#](/stochastic-population-model-fsharp/), [Rust](/stochastic-population-model-rust/)). I ended that Rust post by noting that I was interested in Rust as a modern alternative for scientific computing and thought it would be a good learning exercise to re-write small legacy Fortran programs in Rust. In the process of looking for Fortran programs to translate to Rust, though, I found myself becoming more interested in the idea of learning Fortran than Rust, particularly after learning about the efforts to improve the tooling around Fortran (e.g., [here](https://ondrejcertik.com/blog/2021/03/resurrecting-fortran/) and [here](https://youtu.be/JUHS-JFvs90)). So, here we are...exploring Fortran via the [stochastic population model exercise](https://github.com/hinkelman/stochastic-population-model). 

<!-- more -->

### Standalone Fortran Program

First, I wrote a [standalone Fortran program](https://github.com/hinkelman/stochastic-population-model/blob/main/stochastic-logistic.f90) of the stochastic population model. The main thing that tripped me up was reading arguments from the command line. I initialized a 1D array for holding the values of the arguments. 

```
character(len=20), dimension(5) :: args
```

I initially missed the `len` argument for `character` and was confused by the results I was getting until I realized that the program was only reading the first character provided for each argument. 

We can straightforwardly fill the `args` array with a loop.

```
do i=1,command_argument_count()
  call get_command_argument(i, args(i))
end do
```

But the syntax for converting the arguments from strings to the correct types is unusual. 

```
read(args(1), *) t
```

`t` is the first value in `args` (indexed with `args(1)`). The `*` indicates the default format of the value, which the compiler infers is an `integer` because that is the type of `t`. 

### Fortran Subroutines

I also created a [file](https://github.com/hinkelman/stochastic-population-model/blob/main/stochastic-logistic-subroutines.f90) that only contains subroutines for calling from R. Two of the subroutines (`rstduniform` and `rnormal`) are mostly the same as from the main program, but the names are changed because the R help for the `.Fortran` function recommended not using underscores in the subroutine names and the type of several parameters is `double precision`, not `real`, for compatibility with R. One of the subroutines, `rnorm`, is only included as a test of `rnormal`. 

```
subroutine rnorm(reps, mu, sigma, arr)
  implicit none
  integer :: i
  integer, intent(in) :: reps
  double precision, intent(in) :: mu, sigma
  double precision, intent(inout) :: arr(reps)
  do i=1,reps
    call rnormal(mu, sigma, arr(i))
  end do
end subroutine rnorm
```

The `arr` parameter is declared as `intent(inout)` because on the R side we will pass in a vector that is filled on the Fortran side and then returned again to R. There are performance pitfalls with this approach if you are passing large vectors back-and-forth between Fortran and R, but those can be partly mitigated with the use of the [dotCall64 package](https://cran.r-project.org/web/packages/dotCall64/index.html).

We compile this file into a shared object for use in R with 

```
R CMD SHLIB stochastic-logistic-subroutines.f90
```

### Calling Fortran from R

In the [R code](https://github.com/hinkelman/stochastic-population-model/blob/main/stochastic-logistic.R), we load the shared object with

```
dyn.load("stochastic-logistic-subroutines.so")
```

To test that the `rnormal` subroutine is giving reasonable results, we call the `rnorm` subroutine as the first argument to `.Fortran`. In this example, we draw 1 million random variates from a normal distribution with a mean of 7.8 and standard deviation of 8.3.

```
> n = 1000000L
> rnorm_result <- .Fortran("rnorm", n, 7.8, 8.3, vector("numeric", n))
# .Fortran returns a list; 4th element of list in this case contains the vector
> mean(rnorm_result[[4]])
[1] 7.798748
> sd(rnorm_result[[4]])
[1] 8.302705
```

Following the original blog post, we benchmark the R, Rcpp, and Fortran versions. 

```
> library(microbenchmark)
> microbenchmark(
    Rcpp = logmodc(t, yinit, r, k, thetasd),
    Fortran = .Fortran("logmodf", t, yinit, r, k, thetasd, vector("numeric", t)),
    R = logmodr(t, yinit, r, k, thetasd),
    times = 500L
  )
Unit: microseconds
    expr    min      lq     mean  median      uq     max neval
    Rcpp 13.235 14.6260 17.11903 15.5280 16.8100  86.612   500
 Fortran 21.950 22.8965 26.76053 23.8760 25.3160  72.528   500
       R 56.994 58.8775 67.25860 59.9085 62.3655 243.965   500
```

The original blog post (from 2017) observed a 19x speed up of the Rcpp version over the R version. Here we only get a 4x improvement presumably related to [performance](https://blog.revolutionanalytics.com/2017/02/preview-r-340.html) [improvements](https://blog.revolutionanalytics.com/2018/04/r-350.html) in R. Our Fortran version is 2.5x faster than the R version. 

Again, following the original blog post (with a slight modification), we benchmark running multiple simulations.

```
> rseq <- seq(1.1, 2.2, length.out = 10)
> microbenchmark(
    Rcpp = lapply(rseq, function(x) logmodc(t, yinit, x, k, thetasd)),
    Fortran = lapply(rseq, function(x) .Fortran("logmodf", t, yinit, x, k, thetasd, vector("numeric", t))),
    R = lapply(rseq, function(x) logmodr(t, yinit, x, k, thetasd)),
    times = 500L
  )
Unit: microseconds
    expr     min       lq     mean   median       uq      max neval
    Rcpp 159.578 166.4295 214.4782 198.2240 202.9475 8265.555   500
 Fortran 233.397 239.9620 264.1335 256.8515 262.1665 2994.857   500
       R 621.029 635.1115 676.0114 665.7525 674.2735 2952.635   500
```

The relative performance gains for multiple simulations are similar to the single simulation. 

### Conclusion

This post doesn't make a resounding case for using Fortran over C++ for speeding up simulations in R. The C++ code is shorter and runs faster. My C++ avoidance has centered on the complexity of the language. Perhaps there is a simpler subset of C++ that would go a long way towards speeding up my R code. However, [this post](https://www.moreisdifferent.com/2015/07/16/why-physicsts-still-use-fortran/) makes the case that Fortran is still easier to learn than C++ for physics simulations.

