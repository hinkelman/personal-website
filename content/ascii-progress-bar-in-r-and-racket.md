+++
title = "ASCII progress bar in R and Racket"
date = 2019-07-17
[taxonomies]
tags = ["R", "Racket"]
+++

In a [previous post](/post/progress-bar-widget-in-r-and-racket/), I used GUI toolkits to make progress bars in R and Racket. However, I usually prefer the ASCII progress bars of the [`progress` package](https://github.com/r-lib/progress) in R. The `progress` package includes several options for formatting the progress bar. I particularly like the option to display the estimated time remaining. However, for this post, we will stick to the basic progress bar.

<!-- more -->

```
library(progress)

pb <- progress_bar$new(total = 100)
for (i in 1:100) {
  pb$tick()
  Sys.sleep(0.05)
}

[==============================>-------------------------------]  50%
```

The basic progress bar only requires specifiying the total number of iterations. A default label is produced automatically. The progress bar is customizable (messages, characters, etc.) and works from the command line, Emacs, and RStudio.

My attempt to create an ASCII progress bar in Racket falls **far** short of the capabilities of the `progress` package [[1]](#1). My Racket ASCII progress bar uses the functionality provided by the [`raart` package](https://docs.racket-lang.org/raart/), which I also used to format the tabular output of a [microbenchmarking function](/post/microbenchmarking-in-r-and-racket/).

```
#lang racket

(require raart)

(define (generate-bar value)
  (when (or (< value 0) (> value 100))
    (error "given value is not between 0 and 100"))
  (unless (integer? value)
    (error "given value is not an integer"))
    
  ;; scale progress bar by half to fit in default terminal window on mac (80 columns) 
  (define new-value (floor (/ value 2)))
  (define num-equals (if (= new-value 0) 0 (- new-value 1)))
  ;; always one arrow character unless value is zero
  (define arrow (if (= new-value 0) "" ">"))
  
  ;; placeholder-string to maintain constant width as progress changes
  ;; "  0%", " 10%", "100%"
  (define placeholder-string  
    (cond
      [(< value 10) "  "]
      [(< value 100) " "]
      [else ""]))
      
  (text (string-append
         "["
         (make-string num-equals #\=)
         arrow
         (make-string (- 50 new-value) #\-)
         "]"
         placeholder-string
         (~a value)
         "%")))
```

As a first step, we create a function, `generate-bar`, that appends the characters of the progress bar into a single string. The basic idea is straightforward and facilitated by `make-string` ([`#\`](https://docs.racket-lang.org/reference/reader.html#%28part._parse-character%29) starts a character constant).

```
> (make-string 15 #\-)
"---------------"
> (make-string 0 #\-)
""
```

I flailed around for too long trying to work out the logic for appending the strings. When I took a step back and made the table below, the logic in `generate-bar` became clear. 

<img src="/img/ascii-progress-table.png" width="50%"/>

The `text` function converts the appended string to an `raart` object. The `raart` object can be drawn to a fresh buffer with `draw-here`. 

```
> (draw-here (generate-bar 0))
[--------------------------------------------------]  0%
> (draw-here (generate-bar 1))
[--------------------------------------------------]  1%
> (draw-here (generate-bar 2))
[>-------------------------------------------------]  2%
> (draw-here (generate-bar 50))
[========================>-------------------------] 50%
> (draw-here (generate-bar 98))
[================================================>-] 98%
> (draw-here (generate-bar 99))
[================================================>-] 99%
> (draw-here (generate-bar 100))
[=================================================>]100%
```

[`make-cached-buffer`](https://docs.racket-lang.org/raart/index.html?q=make-cached-buffer#%28def._%28%28lib._raart%2Fbuffer..rkt%29._make-cached-buffer%29%29) creates a buffer (1 row x 70 columns) and allows for the progress bar in the buffer to be updated [[2]](#2). When the loop finishes, `newline` places the cursor on the next line instead of at the end of the progress bar.

```
(define buffer (make-cached-buffer 1 70))
(for ([i (in-range 0 101)])
  (sleep 0.05)
  (draw buffer (generate-bar i)))
(newline)
```

I was able to produce a minimal working ASCII progress bar, but it only works from the command line [[3]](#3). Because Racket programs are generally faster when run from the command line than through DrRacket, this is probably not a limitation. If you have a long-running program that you want to monitor with a progress bar, then you will probably want to run it from the command line anyway. 

Ultimately, I'm not very satisfied with my ASCII progress bar in Racket. But this exercise has given me a new appreciation for the excellence of the R `progress` package and Racket's GUI toolkit.

***

<a name="1"></a> [1] In trying to make a progress bar for Chez Scheme, I realized that Racket's `raart` package was overkill for this task. See [this post](/posts/ascii-progress-bar-chez-scheme/) for my updated approach with code that works in Chez Scheme and Racket.

<a name="2"></a> [2] Using `draw-here` in the loop would draw 101 progress bars.

<a name="3"></a> [3] You can run Racket files from the command line with, for example, `racket progress.rkt` after changing the directory (or specifying the full path to the file).

