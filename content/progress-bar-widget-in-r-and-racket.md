+++
title = "Progress bar widget in R and Racket"
date = 2019-07-13
[taxonomies]
tags = ["R", "Racket", "Tk", "GUI"]
+++

As an impatient person and an insecure programmer, I typically use progress bars for any code that takes more than a few minutes to run. In R, a progress bar widget is available through the [`tcltk` package](https://stat.ethz.ch/R-manual/R-devel/library/tcltk/html/tcltk-package.html). 

<!-- more -->

```
library(tcltk)

pb <- tkProgressBar(title = "Progress Bar", label = "0%", max = 100)
for(i in 1:100) {
  Sys.sleep(0.05)
  setTkProgressBar(pb, value = i, label = paste0(i, "%"))
}
close(pb)
```

In this code, we initialize a progress bar and assign it to `pb`. As the loop progresses, updates are sent to the progress bar that change the value and label of the progress bar. When the loop is complete, the progress bar is closed. On macOS [[1]](#1), the progress bar looks like this:

<img src="/img/progress-r.png" width="336" height = "101">

In Racket, we can build a progress bar widget using the [GUI toolkit](https://docs.racket-lang.org/gui/) [[2]](#2). In my first try, I was able to write code (not shown here) to create the progress bar, but I struggled to see how to make it easily reusable. As usual, I reached out to the [Racket mailing list](https://groups.google.com/forum/#!topic/racket-users/qKijYKGdo4U). After some gentle nudging from the mailing list, I realized that it was straightforward to return a GUI object from a function (also not shown here but see mailing list link). However, another mailing list member provided an alternative implementation based on defining a new class, which I think is the more natural way to approach this task.

```
#lang racket/gui

(define progress-bar%
  (class horizontal-pane%
    (super-new)

    (define gauge (new gauge%
                       [label #f]
                       [parent this]
                       [range 100]))

    ;; initialize to 100% with no auto resize
    ;; this way the label has the right size
    ;; so that everything can be seen
    ;; and it does not "wobble" around while filling up
    (define msg (new message%
                     [parent this]
                     [label "100%"]))

    (define/public (set-value val)
      (send gauge set-value val)
      (send msg set-label (string-append (~a val) "%")))

    ;; set back to 0%
    (set-value 0)))
```

In this code, a new class, `progress-bar%`, is defined as a [`horizontal-pane%`](https://docs.racket-lang.org/gui/horizontal-pane_.html?q=set-label) that contains a [`gauge%`](https://docs.racket-lang.org/gui/gauge_.html?q=set-label) and a [`message%`](https://docs.racket-lang.org/gui/message_.html?q=set-label). The arrangement of the `gauge%` and `message%` is determined by the order that they appear in the `progress-bar%` definition (i.e., left to right in a `horizontal-pane%`).

The `progress-bar%` class includes a single method, `set-value`, for updating both the `gauge%` and the `message%` [[3]](#3). Interestingly, you can call the `set-value` method from within the class definition. 

```
(define frame (new frame%
                   [label "Progress Bar"]
                   [width 300]))

(define progress (new progress-bar% [parent frame]))

(send frame show #t)
(for ([i (in-range 1 101)])
  (sleep 0.05)
  (send progress set-value i))
(send frame show #f)
```

A [`frame%`](https://docs.racket-lang.org/gui/frame_.html?q=set-label) is defined as a top-level container for `progress`. The `frame` is shown or hidden by sending `#t` or `#f`, respectively, with the `show` method. As in R, the value of the progress bar and label is updated on each iteration. On macOS, the progress bar looks like this:

<img src="/img/progress-racket.png" width="297" height = "45">

***

<a name="1"></a> [1] [XQuartz](https://www.xquartz.org) is required to display the widget on macOS.

<a name="2"></a> [2] For more on building GUI's in Racket, see also [Alex Harsányi's blog](https://alex-hhh.github.io/index.html).

<a name="3"></a> [3] `~a` is shorthand for converting a number to a string. `paste` in R implicitly converts values to strings before concatenation, but `string-append` in Racket requires that all arguments are strings.