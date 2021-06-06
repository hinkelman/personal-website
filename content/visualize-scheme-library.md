+++
title = "Visualizing Scheme library procedures with an interactive network graph in R"
date = 2021-06-05
[taxonomies]
categories = ["Chez Scheme", "dataframe", "R", "Shiny"]
tags = ["dataframe", "visNetwork", "pkgnet"]
+++

As a learning exercise, I wrote a [`dataframe`](https://github.com/hinkelman/dataframe/) library for Scheme. Because I was learning Scheme while I wrote `dataframe`, I did not prioritize performance. However, as I've tried to use the `dataframe` library ([exploratory data analysis](/eda-chez-scheme), [spam simulation](/spam-simulation-chez-scheme/), [gapminder](/gapminder-base-r-scheme/)), I've encountered performance pitfalls that make `dataframe` largely unusable for datasets with more than a few thousand rows. I have a rough idea of where the bottlenecks are, but I thought it would be a useful to take a step back and visualize the `dataframe` procedures as a network graph. 

<!-- more -->

I had [some experience](https://twitter.com/travishinkelman/status/1202359425635241984) with the R package, [`pkgnet`](https://uptake.github.io/pkgnet/), which allows for exploring the structure of a package by building a graph representation. I had also spent a little time using the package, [`visNetwork`](https://datastorm-open.github.io/visNetwork/), that `pkgnet` uses to build the function graph. Moreover, because code is data in Scheme, it is relatively straightforward to analyze Scheme code and I had briefly experimented with that in a [previous blog post](/viewing-source-code-r-chez-scheme). 

## Prepare Data

All of the Scheme code to analyze the `dataframe` procedures is found [here](https://github.com/hinkelman/dataframe/blob/master/network-graph/network-graph.ss). Below I will walk through the main ideas. 

First, let's create a silly example of Scheme library code. `example` is the list that would be created as the result of reading a file called `example-library.sls`.

```
(define example
  '(library (example-library)
     (export exported-proc)
     (import (rnrs))
     (define exported-proc
       (case-lambda
         [(x1) (exported-proc-helper x1 10)]
         [(x1 x2) (exported-proc-helper x1 x2)]))
     (define (exported-proc-helper x1 x2)
       (let ([x-sum (sum2 x1 x2)])
         (map add1 (iota x-sum))))
     (define (sum2 x1 x2)
       (+ x1 x2))
     (define (add1 x)
       (+ x 1))
     (define (iota count)
       (define start 0)
       (define step 1)
       (let loop ((n 0) (r '()))
         (if (= n count)
	     (reverse r)
	     (loop (+ 1 n)
	           (cons (+ start (* n step)) r)))))))
```

We can work with `example` in the same way as any other Scheme list.

```
> (car example)
library
> (length example)
9
```

The following two procedures are used to extract the procedure names from `example`.

```
;; get all procedure definitions
(define (get-defs lst)
  (filter (lambda (x) (and (pair? x) (symbol=? (car x) 'define))) lst))

;; name is the procedure name
;; def is one element of the list from get-defs
(define (get-name def)
  (if (pair? (cadr def))
      (caadr def)
      ;; cadr version for definitions using lambda or case-lambda
      (cadr def)))
```

We map `get-name` across the list from `get-defs` to get our list of procedure names.

```
> (define defs (get-defs example))
> (define names (map get-name defs))
> names
(exported-proc exported-proc-helper sum2 add1 iota)
```

The procedure names are the nodes in our network graph. The connections between the procedures are the edges. `visNetwork` requires that the edge list is defined by ID numbers, not procedure names. Next we create a list of pairs with each procedure assigned an ID number.

```
> (define names-nums 
      (map (lambda (name num) (cons name num)) names (enumerate names)))
                                                          
> names-nums
((exported-proc . 0)
  (exported-proc-helper . 1)
  (sum2 . 2)
  (add1 . 3)
  (iota . 4))
```
Now that we have enumerated our procedures, we need to iterate through all of the procedure definitions to identify which other procedures are called from within each procedure. Even though recursion is great for working with nested data structures, the `get-edges` procedure is the first time that I had ever used deep recursion (i.e., recursing on both the `car` and `cdr` of a list).

```
(define (get-edges def names-nums)
  (let* ([name (get-name def)]
         [num (cdr (assoc name names-nums))]
         [out (let loop ([body (cddr def)]
                         [results '()])
                (cond [(null? body)
                       results]
                      [(not (pair? body))
                       (let ([name-num (assoc body names-nums)])
                         (if name-num (cons (cdr name-num) results) results))]
                      [else
                       (loop (car body) (loop (cdr body) results))]))])
    (map (lambda (x) (cons num x)) (remove-duplicates out))))
```

`get-edges` returns a list of pairs where the `car` is the ID of `def` and the `cdr` is the ID of the procedures called by `def`. Here is the output of `get-edges` when applied to `exported-proc-helper`:

```
> (cadr defs)
(define (exported-proc-helper x1 x2)
  (let ([x-sum (sum2 x1 x2)]) (map add1 (iota x-sum))))
> (get-edges (cadr defs) names-nums)
((1 . 2) (1 . 3) (1 . 4))
```

That covers the main ideas in preparing the data. The rest of the [code](https://github.com/hinkelman/dataframe/blob/master/network-graph/network-graph.ss) just applies those ideas across multiple files and writes the data for use by R.

## Visualize Data

The network graph is visualized in a [Shiny](https://shiny.rstudio.com/) app with `visNetwork`. The code for the app can be found in [this gist](https://gist.github.com/hinkelman/df2422122a4a0588973dd2af443a1100). The live app can be viewed [here](https://hinkelman.shinyapps.io/dataframe-network-graph/). [Note, it takes several seconds to build and display the graph in the Shiny app.] It requires remarkably little code to visualize the network graph. For example, this is all that is required for the server code in the Shiny app.

```
output$networkGraph <- renderVisNetwork({
    nodes %>% 
        visNetwork(edges) %>%
        visEdges(arrows = "to") %>% 
        visOptions(highlightNearest = TRUE, 
                    nodesIdSelection = TRUE)
})
```

## Conclusions

I haven't yet spent much time investigating the network graph for the `dataframe` procedures and I have only a few observations.

* I almost always use `named let` for recursion so there are very few nodes with arrows looping back to themselves (see `alist-modify-loop` as an exception).
* Unsurprisingly, a few procedures are called by a lot of other procedures, e.g., `make-dataframe`, `dataframe-alist`, `check-dataframe`.
* A few procedures (`make-dataframe`, `dataframe-alist`, `dataframe-dim`) are created automatically as part of defining the dataframe record type. I added them to the list of procedure names by appending the list of exported procedure names to the list of extracted procedure names (and removing duplicates from the appended list). I think the only connection that is lost in this approach is that `check-alist` is called from `make-dataframe` but that is not reflected in the graph.
