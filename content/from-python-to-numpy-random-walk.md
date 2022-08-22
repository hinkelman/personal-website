+++
title = "From Python to NumPy: random walk example in R and Chez Scheme"
date = 2021-08-21
[taxonomies]
categories = ["Python", "R", "Chez Scheme"]
tags = ["from-python-to-numpy", "vectorization", "timing"]
+++

As a learning exercise, I decided to translate examples from the book, [From Python to NumPy](https://www.labri.fr/perso/nrougier/from-python-to-numpy/), into R and Chez Scheme. This post describes the [random walk example from Chapter 2](https://www.labri.fr/perso/nrougier/from-python-to-numpy/#simple-example). All of the code is in [this repository](https://github.com/hinkelman/from-python-to-numpy) so I will only highlight a few pieces of code below. For context, I am a long-time R programmer who only periodically pokes at Python and dabbles in Scheme for fun. Because performance is the primary motivation of vectorizing code with NumPy in Python, I will be loosely comparing timings between Python, R, and Chez Scheme. Take these timings with a large grain of salt. I don't know how comparable the different timings are.

<!-- more -->

The simple example to motivate the book involves a 1D random walk. The book starts with a `for loop` example, reports a 7x speed up over the `for loop` by vectorizing the code with `itertools`, and reports a 500x improvement from `for loop` to NumPy (but, by my math, the reported timings show 1000x). On my machine, I also observe a 7x speed up from `for loop` to `itertools` but only a 80x jump from `for loop` to NumPy. The `for loop` in R has comparable peformance to the `for loop` in Python, but vectorized R was about 2x as fast as NumPy. Here are vectorized versions of the functions in Python (NumPy) and R:

Python
```
def random_walk_fastest(n=1000):
    steps = np.random.choice([-1,+1], n)
    return np.cumsum(steps)
```

R
```
random_walk_v <- function(n = 1000) {
  steps <- sample(c(-1, 1), size = n, replace = TRUE)
  cumsum(steps)
}
```

Chez Scheme doesn't have the option to speed up code by vectorizing it, but Chez is known as one of the most performant Scheme implementations. I tried a couple of versions of the procedure in Chez. In one, I iterated over a vector with a `do loop`. In the other, I used recursion on a list. Both performed similarly at about 1.5x as fast as the vectorized R version. Here is the recursive version:

```
(define (random-help)
  ;; (random 2) returns 0 or 1
  (- (* 2 (random 2)) 1))

(define (random-walk-lst n)
  (let loop ([step 0]
             [position (random-help)]
             [walk '()])
    (if (= step n)
        (reverse walk)
        (loop (add1 step)
              (+ position (random-help))
              (cons position walk)))))
```

In the next example, the author makes the point that the best NumPy performance often comes at the expense of readability. The example involves returning the starting index for all occurrences of a sub-sequence that are found in the random walk list. The book indicates that the NumPy version was 10x faster than pure Python (7x on my machine). In trying to identify the best way to implement this in R, I found this [mailing list thread](https://stat.ethz.ch/pipermail/r-help/2012-February/303756.html) with numerous solutions that vary widely in speed. The fastest R version (of the ones that I tried) was nearly 8x faster than the next best solution. I implemented that same algorithm in Python (+ a little NumPy). It was nearly 4x as fast as the NumPy example from the book and comparably fast to the R version. Here is that algorithm in Python and R (apologies for the inconsistent naming between my R and Python files):

Python
```
def find_crossing_3(seq, sub):
    n = len(seq)
    m = len(sub)
    candidate = np.arange(n-m)
    for i in range(m):
        candidate = candidate[sub[i] == seq[candidate + i]]
    return candidate
```

R
```
find_crossing_2 <- function(seq, sub) {
  n <- length(seq)
  m <- length(sub)
  candidate <- seq_len(n - m + 1)
  for (i in seq_len(m)) {
    candidate <- candidate[sub[i] == seq[candidate + i - 1]]
  }
  candidate
}
```

Unfortunately, it is not possible (as far as I understand) to reproduce this algorithm in Chez Scheme and my alternative was about 50x slower than the Python and R versions.

```
;; https://stackoverflow.com/a/28034455
(define (first-n lst n)
  (if (zero? n)            
      '()                
      (cons (car lst)         
            (first-n (cdr lst)    
                     (- n 1)))))

(define (find-crossing-lst seq sub)
  (let loop ([index 0]
             [lst seq]
             [results '()])
    (if (< (length lst) (length sub))
        (reverse results)
        (if (equal? (first-n lst (length sub)) sub)
            (loop (add1 index) (cdr lst) (cons index results))
            (loop (add1 index) (cdr lst) results)))))
```

After asking on the [Scheme Discord server](https://discord.gg/8zjfdtj4), it was pointed out that the code above re-calculates the length on each iteration. Making the minor change below leads to a 16x improvement. Still over 3x slower than R and Python, but likely still room for optimizations.

```
(define (find-crossing-lst seq sub)
  (let ([seq-len (length seq)]
        [sub-len (length sub)])
    (let loop ([index 0]
               [lst seq]
               [results '()])
      (if (< (- seq-len index) sub-len)
          (reverse results)
          (if (equal? (first-n lst sub-len) sub)
              (loop (add1 index) (cdr lst) (cons index results))
              (loop (add1 index) (cdr lst) results))))))
```