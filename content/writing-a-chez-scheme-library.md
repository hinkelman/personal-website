+++
title = "Writing a Chez Scheme library"
date = 2019-09-11
[taxonomies]
tags = ["chez-stats", "Chez Scheme"]
+++

Recently, I [switched](/post/exploring-scheme-implementations/) from learning Racket to Chez Scheme. I wanted to try to repeat some of my previous Racket exercises in Chez Scheme, but quickly ran into a barrier when my [first choice](/post/stochastic-population-model-r-racket/) required drawing random variates from a normal distribution. I looked for existing Chez Scheme libraries but came up empty. I considered [SRFI 27: Sources of Random Bits](https://srfi.schemers.org/srfi-27/srfi-27.html), which includes example code for generating random numbers from a normal distribution, and [reached out for guidance](https://www.reddit.com/r/scheme/comments/cnw0cy/generating_random_variates/). Ultimately, I decided that it would be a good exercise to write a library for generating random variates from different distributions. As I started to write the random variate procedures, I realized that I minimally needed procedures for calculating mean and variance to test the output of the random variate procedures. And, thus, the scope of the library started to expand and the [`chez-stats` library](https://github.com/hinkelman/chez-stats) was born.

<!-- more -->

Writing `chez-stats` has been a great learning experience. Even though I have lots of applied statistics experience, I had no idea that [accurately calculating sample variance is challenging](https://www.johndcook.com/blog/standard_deviation/) or that there are nine algorithms to choose from when using [`quantile` in R](https://stat.ethz.ch/R-manual/R-devel/library/stats/html/quantile.html) [[1]](#1) (and also in `chez-stats`). 

When I was [choosing a new programming language to learn](/post/programming-horizons/), I did not make the size of the package ecosystem a key consideration, but, in retrospect, I think it was clearly a factor in choosing Racket. And, as I started to switch my attention to Chez Scheme, I had several moments where I almost went running back to Racket when faced with the lack of third-party libraries for Chez Scheme. I'm not keen on reinventing the wheel, but I previously underestimated how valuable that experience could be [[2]](#2). As I gain experience with Chez Scheme, it will be interesting to see if I continue to value writing my own libraries [[3]](#3) or start to lament the lack of libraries.

When I started working on `chez-stats`, I made a couple of decisions to simplify my efforts. For one, I didn't make any effort to write portable scheme code. Second, I stuck to the list as the primary data structure. It would be nice to have the flexibility to also work with vectors, but I am kicking that can down the road.

When writing the procedures in `chez-stats`, I primarily consulted [R](https://www.r-project.org) source code for `statistics` and [slides by Raj Jain](https://www.cse.wustl.edu/~jain/books/ftp/ch5f_slides.pdf) for `random-variates`. I used [SRFI 64: A Scheme API for Test Suites](https://srfi.schemers.org/srfi-64/srfi-64.html) to write a test suite for `chez-stats`. In over ten years of writing R code, I have never written any tests. It's definitely more tedious and less fun than writing the core `chez-stats` procedures, but I find it very satisfying to see the test suite run without any failures. I used markdown to write some basic documentation in the README hosted on the GitHub repo. By far, writing documentation is my least favorite part of this whole process.

As of right now, I have no plans to expand the functionality of `chez-stats`. My next steps will be to try to put `chez-stats` to work for me and, in the process, identify friction points and missing features. 

***

<a name="1"></a> [1] I had only ever used the default value.

<a name="2"></a> [2] And **vastly** underestimated how satisfying that experience could be.

<a name="3"></a> [3] [The Lisp Curse](http://www.winestockwebdesign.com/Essays/Lisp_Curse.html)