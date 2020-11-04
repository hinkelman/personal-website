+++
title = "Viewing source code in R and Chez Scheme"
date = 2020-04-22
[taxonomies]
categories = ["Chez Scheme", "R"]
tags = ["methods", "getAnywhere", "inspect", "introspection", "read", "filter"]
+++

One of the advantages of open source software is being able to view, review, and learn from source code. Both R and Chez Scheme provide tools for accessing source code.

<!-- more -->

### R

[`cfs.misc`](https://github.com/fishsciences/cfs.misc) is a small package with a few utility functions that are useful to our work at [Cramer Fish Sciences](https://www.fishsciences.net). One of the functions in `cfs.misc`, `water_year`, when given a date, returns the [water year](https://en.wikipedia.org/wiki/Water_year) (as defined by the USGS). After installing `cfs.misc`, you can view the source code by simply typing the function name in the console [[1]](#1) (here prefixed by the package name instead of loading the package via `library`).

```
> cfs.misc::water_year 
function (x) 
{
    x_lt <- as.POSIXlt(x)
    x_lt$year + 1900L + ifelse(x_lt$mon + 1L >= 10L, 1L, 0L)
}
<bytecode: 0x7f9e33e3f980>
<environment: namespace:cfs.misc>
```

`cfs.misc::water_year` is based on `lubridate::year`. Let's look at the source code for `lubridate::year`.

```
> lubridate::year
function (x) 
UseMethod("year")
<bytecode: 0x7f9e316ee8d0>
<environment: namespace:lubridate>
```

Encountering `UseMethod("year")` tells us that `lubridate::year` is a generic function and the method used depends on the object passed to the generic function. Let's see what methods are available.

```
> methods(lubridate::year)
[1] year.default* year.Period* 
see '?methods' for accessing help and source code
```

`year.default` looks promising.

```
> lubridate::year.default
Error: 'year.default' is not an exported object from 'namespace:lubridate'
```

That makes sense. You won't find `year.default` on the [`lubridate` reference page](https://lubridate.tidyverse.org/reference/index.html). The triple-colon operator, `:::`, allows us to use unexported functions.

```
> lubridate:::year.default("2020-04-22")
[1] 2020
```

We can also use `:::` to view source code for unexported functions.

```
> lubridate:::year.default
function (x) 
as.POSIXlt(x, tz = tz(x))$year + 1900
<bytecode: 0x7f9e37ecd9f0>
<environment: namespace:lubridate>
```

Alternatively, you could use `getAnywhere`.

```
> getAnywhere("year.default")
A single object matching ‘year.default’ was found
It was found in the following places
  namespace:lubridate
with value

function (x) 
as.POSIXlt(x, tz = tz(x))$year + 1900
<bytecode: 0x7f9e37ecd9f0>
<environment: namespace:lubridate>
```

Next, let's look at `dplyr::select` as a more complex example. 

```
> methods(dplyr::select)
[1] select.data.frame* select.default*    select.grouped_df* select.list        select.tbl_cube*  
see '?methods' for accessing help and source code

> dplyr:::select.data.frame
function (.data, ...) 
{
    vars <- tidyselect::vars_select(tbl_vars(.data), !!!enquos(...))
    select_impl(.data, vars)
}
<bytecode: 0x7f9e32a9aa68>
<environment: namespace:dplyr>

> dplyr:::select_impl
function (df, vars) 
{
    .Call(`_dplyr_select_impl`, df, vars)
}
<bytecode: 0x7f9e39c12360>
<environment: namespace:dplyr>
```

As we drill down into the source code, we eventually hit `.Call`, which indicates that compiled code from another language is being called. We can go no farther from R and would need to seek out the source file for the compiled code.

In contrast, `poorman::select` [[2]](#2) provides similar functionality as `dplyr::select`, but the source code is written in base R and, thus, is more easily accessible.

```
> poorman::select
function (.data, ...) 
{
    map <- names(deparse_dots(...))
    col_pos <- select_positions(.data, ..., group_pos = TRUE)
    res <- extract(.data, , col_pos, drop = FALSE)
    to_map <- nchar(map) > 0L
    colnames(res)[to_map] <- map[to_map]
    if (has_groups(.data)) 
        res <- set_groups(res, group_vars(.data))
    res
}
<bytecode: 0x7f9e35bd14d0>
<environment: namespace:poorman>
```

I've only shown examples from R packages, but the same techniques also work with base R code. Everything that I've covered so far is better explained in _R Help Desk: Accessing the Sources_ by Uwe Ligges in [this issue](https://cran.r-project.org/doc/Rnews/Rnews_2006-4.pdf) of _R News_.

### Chez Scheme

In Chez Scheme, we can view source code for exported procedures with the [inspector](https://cisco.github.io/ChezScheme/csug9.5/debug.html#./debug:s22). Let's load up my [`chez-stats` library](https://github.com/hinkelman/chez-stats) and inspect `mean` [[3]](#3).

```
> (import (chez-stats))

> (inspect mean)
#<procedure mean at statistics.ss:3309>                           : c
(lambda (ls) ((...) ls ...) (/ (...) ...))                        : p

(lambda (ls)
  (($top-level-value
     '#{check-list iyij3kx7j76i8m1qgvaikyqyy-2})
    ls
    "ls"
    "(mean ls)")
  (/ (apply + ls) (length ls)))

(lambda (ls) ((...) ls ...) (/ (...) ...))                        : q
```

The inspector is an interactive tool. To see the available commands, type `?` after the `:` prompt. Type `q` to quit the inspector. In the example above, I first ask for the code with `c` and then type `p` for pretty printing of the code. 

`mean` calls `check-list` to check if the input `ls` is a list. `check-list` is not exported from `chez-stats` and is not availble to the inspector. 

```
> (inspect check-list)
Exception: variable check-list is not bound
```

However, because I've set up assertion procedures, like `check-list`, as part of a library, we can still view `check-list` with the inspector.

```
> (import (chez-stats assertions))

> (inspect check-list)
#<procedure check-list at assertions.ss:227>                      : c
(lambda (ls ls-name who) (if (...) ...) (if (...) ...) ...)       : p

(lambda (ls ls-name who)
  (if (not (list? ls))
      (assertion-violation who
        (string-append ls-name " is not a list"))
      (#2%void))
  (if (not (for-all real? ls))
      (assertion-violation who
        (string-append
          "at least one element of "
          ls-name
          " is not a real number"))
      (#2%void))
  (if (null? ls)
      (assertion-violation who
        (string-append ls-name " is empty"))
      (#2%void)))
```

The code returned by the inspector is expanded. All of the `if` statements in `check-list` were written as `unless` or `when`, but expanded to `if` with a dead branch when shown by the inspector. I can see how this could be useful for better understanding how Chez works, but, generally, if I'm digging around in source code, I would prefer to see the more human-readable version. 

I only recently learned about the inspector from the Chez Scheme [mailing list](https://groups.google.com/forum/#!topic/chez-scheme/x6auPRuweEs). I'm happy to have it in my toolbelt, but a little disappointed that it only works for procedures exported from external libraries, not for unexported procedures or Chez's built-in procedures. In the case of my [`dataframe` library](https://github.com/hinkelman/dataframe/), for example, most of the exported procedures are simple wrappers around layers of unexported procedures, which limits the utility of the inspector.

It eventually occurred to me, though, that Chez maybe doesn't need to provide a lot of built-in tools for inspecting code because, with s-expressions, it is very easy to inspect the code yourself. For example, if you have downloaded my `dataframe` library, then we can read the source code as a list.

```
(define df-code (with-input-from-file "/path/to/df.ss" read))
```

Next, we write procedures to filter `df-code` for elements that start with `define` and search for a specified procedure within that list of `define` elements. 

```
(define (get-define ls)
  (filter (lambda (x) (and (pair? x) (symbol=? (car x) 'define))) ls))
  
(define (find-proc ls proc-symbol)
  (filter (lambda (x) (and (pair? (cadr x)) (symbol=? (caadr x) proc-symbol))) ls))
```

We can use these very simple procedures to quickly view the source code for an unexported procedure, `alist-select`.

```
> (define df-define (get-define df-code))

> (car df-define)
(define (thread-last-helper f value . body)
  (apply f (append body (list value))))

> (find-proc df-define 'alist-select)
((define (alist-select alist names)
   (map (lambda (name) (assoc name alist)) names)))
```

Finally, just to drive home the point that we needed to read the file ourselves...

```
> (import (dataframe df))

> (inspect alist-select)
Exception: variable alist-select is not bound
```

***

<a name="1"></a> [1] If you are using RStudio, you can also view source code with `View(cfs.misc::water_year)`, which opens a tab to show the source code and omits the bytecode information.

<a name="2"></a> [2] The [`poorman` package](https://github.com/nathaneastwood/poorman) attempts to replicate `dplyr` functionality using only base R code. This is not an endorsement of `poorman`. I have never used it myself, but I think it provides an interesting contrast to `dplyr`.

<a name="3"></a> [3] When I tried to use the inspector from the Emacs REPL, it would occassionally hang up. Not sure if that is a quirk of my system or session, but I now run `inspect` after launching Chez from the terminal.
