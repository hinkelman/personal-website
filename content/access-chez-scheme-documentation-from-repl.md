+++
title = "Access Chez Scheme documentation from the REPL"
date = 2020-01-01
updated = 2023-05-02
[taxonomies]
tags = ["chez-docs", "Chez Scheme", "R", "rvest"]
+++

In the process of learning Chez Scheme, I've missed R's ability to quickly pull up documentation from the console via [`help` or `?`](https://www.r-project.org/help.html). I've toyed with the idea of trying to format the contents of the [Chez Scheme User's Guide](https://cisco.github.io/ChezScheme/csug9.5/) for display in the REPL (similar to [Clojure Docs](https://clojuredocs.org/clojure.repl/doc)). But that is probably too big of a task for me at this point. It recently occurred to me, though, that I can write a simple library, [`chez-docs`](https://github.com/hinkelman/chez-docs), with only one procedure, `doc`, that will make it a bit easier to access the Chez Scheme User's Guide.

<!-- more --> 

UPDATE: I overhauled the code behind `chez-docs`. The web scraping is now all done in Chez Scheme (see [here](https://github.com/hinkelman/chez-docs-scrape)), not R, and more of the documentation is displayed from the REPL. 

My typical entry point to learning about Chez Scheme is the [Summary of Forms](https://cisco.github.io/ChezScheme/csug9.5/summary.html) page of the Chez Scheme User's Guide. The simple idea behind `chez-docs` is to scrape [[1]](#1) the data from the Summary of Forms page and write a procedure that opens links to the documentation from the REPL. 

### Web Scraping with R

I used the [`rvest` package](http://rvest.tidyverse.org) for R to scrape the data from the Summary of Forms page. First, I downloaded the page and opened it in a text editor to see how the table was structured. Then, I extracted the URLs by drilling down into the nodes of the HTML document and retrieving the contents of the `href` attribute.

```
library(tidyverse)
library(rvest)

chez_url = "https://cisco.github.io/ChezScheme/csug9.5/summary.html"

chez_links <- read_html(chez_url) %>% 
  html_nodes("table") %>% 
  html_nodes("tr") %>% 
  html_nodes("a") %>% 
  html_attr("href")
```

Next, I retrieved the text contents of the HTML table. `html_table` returns a list with all of the tables on the page as data frames. In this case, there is only one table in the list.

```
chez_table_list <- read_html(chez_url) %>% 
  html_nodes("table") %>% 
  html_table()
```

### Data Preparation with R

The Summary of Forms page links to two sources: [The Scheme Programming Language (TSPL)](https://www.scheme.com/tspl4/) and the [Chez Scheme User's Guide (CSUG)](https://cisco.github.io/ChezScheme/csug9.5/). A `t` in the page number indicates TSPL as the source. The extracted URLs linking to those sources required a little cleanup. 

I'm using `Key` to mean the first 'word' in the `Form` column. In many cases, that 'word' is just a symbol, e.g., `>`, `+`, `*`, etc.

```
chez_table <- chez_table_list[[1]] %>% 
  filter(Form != "") %>%          # drop empty first row
  mutate(URL = chez_links,
         # clean up extracted links to TSPL
         URL = gsub(pattern = "http://scheme.com/tspl4/./",
                    replacement = "https://scheme.com/tspl4/",
                    URL),
         # convert relative to absolute links for CSUG
         URL = gsub(pattern = "^\\.", 
                    replacement = "https://cisco.github.io/ChezScheme/csug9.5", 
                    x = URL),
         Key = sapply(strsplit(Form, "\\s"), "[[", 1),
         Key = gsub("\\(|\\)", "", Key),
         Source = ifelse(substr(Page, 1, 1) == "t", "TSPL", "CSUG")) %>% 
  select(Key, Form, Source, URL) 
```

The problem here is that `Key` is not unique because the same key can be associated with more than one form and/or more than one source. I decided that the simplest solution was to separate the keys by source and combine the forms for each key that shared the same URL. I used nested `for` loops to tear down the data frame and build it back up. 

```
source_list <- list()
excluded_list <- list()
for (j in c("CSUG", "TSPL")){
  ct_source <- filter(chez_table, Source == j)
  key_list <- list()
  excluded <- c()
  for (i in unique(ct_source$Key)){
    ctsk <- filter(ct_source, Key == i)
    if (nrow(ctsk) == 1){
      key_list[[i]] <- ctsk
    } else {
      if (nrow(unique(select(ctsk, Key, Source, URL))) == 1){
        key_list[[i]] <- tibble(Key = i,
                                Form = paste(unique(ctsk$Form), collapse = "~"),
                                Source = j,
                                URL = ctsk$URL[1])
      } else {
        excluded <- c(excluded, i)
      }
    }
  }
  excluded_list[[j]] <- excluded
  source_list[[j]] <- bind_rows(key_list)
}
out <- bind_rows(source_list)
```

I decided that it would look nice to separate the forms with newlines for display in Chez, but writing and reading files with newlines as separators within a column creates a mess. Instead, I chose `~` as the separator in the `Form` column because it is not a character that appears in any of the forms, which makes it easier to replace with `\n` on the Chez side.

I kept track of which keys were excluded to decide if I needed to take additional processing steps. `alias` and `let` were the only keys that were excluded because the two forms are associated with two different links. No additional processing was done to include `alias` and `let`.

The last step in R was to write the processed table to file. Because some of the forms contain commas, e.g., `#,template`, I wrote the table as a TSV file. I split the table into two files because it made the processing simpler in Chez.

```
for (j in c("CSUG", "TSPL")){
  out %>% 
    filter(Source == j) %>% 
    select(-Source) %>% 
    write_tsv(paste0(j, ".tsv"))
}
```

### Data Preparation with Chez Scheme

I used my [`chez-stats` library](https://github.com/hinkelman/chez-stats/blob/master/chez-stats/delimited.sls) to read the tab-delimited files, dropped the header row, and combine the two lists into a list for writing to file.

```
(import (chez-stats))

(define data (list (cons 'csug (cdr (read-delim "R/CSUG.tsv" #\tab)))
                   (cons 'tspl (cdr (read-delim "R/TSPL.tsv" #\tab)))))

(with-output-to-file "chez-docs-data.scm" (lambda () (write data)))
```

### Reading Data in Chez Scheme Library

To read the data when `chez-docs` is loaded, we need to identify the path where the data is located. For `(import (chez-docs))` to work, the user needs to have `chez-docs.ss` and `chez-docs-data.scm` in a directory found by `(library-directories)` [[2]](#2). Thus, we can loop through the list of library directories to find the file location and read the data. 

```
(define data-paths
  (map (lambda (x) (string-append x "/chez-docs-data.scm"))
       (map car (library-directories))))

(define data
  (let ([tmp '()])
    (for-each
     (lambda (path)
       (when (file-exists? path)
         (set! tmp (with-input-from-file path read))))
     data-paths)
    tmp))
```

### Launching Documentation

The main procedure in `chez-docs` is `doc`, which uses `case-lambda` to handle optional arguments with default values. 

```
(define doc
  (case-lambda
    [(proc) (doc-helper proc 'open-link 'both)]
    [(proc action) (doc-helper proc action 'both)]
    [(proc action source) (doc-helper proc action source)]))
```

`data-lookup` checks that the strings passed as arguments are valid and returns a list of the association lists for `proc` from the `data` object created above.

```
(define (data-lookup proc source)
  (cond [(or (symbol=? source 'csug) (symbol=? source 'tspl))
         (let ([result (dl-helper proc source)])
           (if result
               (list result) 
               (assertion-violation "(doc proc action source)"
                                    (string-append proc " not found in " (symbol->string source)))))]
        [(symbol=? source 'both)
         (let ([csug (dl-helper proc 'csug)]
               [tspl (dl-helper proc 'tspl)])
           (if (or csug tspl)
               (list csug tspl)
               (assertion-violation "(doc proc)" (string-append proc " not found in csug or tspl"))))]
        [else
         (assertion-violation "(doc proc action source)" "source not one of 'csug, 'tspl, 'both")]))
         
;; data is imported above
(define (dl-helper proc source)
  (assoc proc (cdr (assoc source data)))) 
```

When using `data-lookup` on `<`, a 2-element list is returned indicating that there is an entry for `<` in both CSUG and TPSL.

```
> (data-lookup "<" 'both)
(("<" "(< real1 real2 real3 ...)"
        "https://cisco.github.io/ChezScheme/csug9.5/numeric.html#./numeric:s67")
  ("<" "(< real1 real2 real3 ...)"
         "https://scheme.com/tspl4/objects.html#./objects:s88"))
```

If `proc` is only found in one source, and both are requested, then one element of the returned list will be `#f`.

```
> (data-lookup "map" 'both)
(#f ("map"
      "(map procedure list1 list2 ...)"
      "https://scheme.com/tspl4/control.html#./control:s30"))
```

`display-form-open` takes a list, `data-selected`, returned by `data-lookup`, displays the form(s), and optionally opens a link to the relevant section of the documentation in your default browser. When `action` is `'open-link`, `display-form-open` makes a system call to `open` (macOS), `xdg-open` (Linux), or `start` (Windows) and requires an internet connection.

```
(define (display-form-open data-selected action)
  (when data-selected
    (display (replace-tilde (string-append (cadr data-selected) "\n")))
    (when (symbol=? action 'open-link)
      (system (string-append open-string (caddr data-selected))))))
```

`(machine-type)` is used to determine the system-specific string, `open-string`, for use in the system call.

```
(define open-string
  (case (machine-type)
    [(i3nt ti3nt a6nt ta6nt) "start "]     ; windows
    [(i3osx ti3osx a6osx ta6osx) "open "]  ; mac
    [else "xdg-open "]))                   ; linux
```

When `action` is `'display-form`, `display-form-open` simply displays the form(s) for the specified `proc`, which is helpful if you can't remember the order of arguments for a procedure.

```
> (display-form-open (car (data-lookup "append" "TSPL")) #f)
(append)
(append list ... obj)
```

For multi-line display of forms, the `~` added in R to separate forms is replaced with `\n` using `replace-tilde`.

```
(define (replace-tilde str)
  (let* ([in (open-input-string str)]
         [str-list (string->list str)])
    (if (not (member #\~ str-list))
        str  ;; return string unchanged b/c no tilde
        (let loop ([c (read-char in)]
                   [result ""])
          (cond [(eof-object? c)
                 result]
                [(char=? c #\~)
                 (loop (read-char in) (string-append result "\n"))]
                [else
                 (loop (read-char in) (string-append result (string c)))])))))
```

The last piece is `doc-helper`, which loops through the output of `data-lookup` and passes it to `display-form-open`.

```
(define (doc-helper proc action source)
  (unless (or (symbol=? action 'open-link)
              (symbol=? action 'display-form))
    (assertion-violation "(doc proc action)" "action not one of 'open-link or 'display-form"))
  (let loop ([ls (data-lookup proc source)])
    (cond [(null? ls) (void)]
          [else
           (display-form-open (car ls) action)
           (loop (cdr ls))])))
```

The downside of this approach is that if a `proc` is found in both sources with the same form, then it will be displayed twice. I decided this behavior isn't sufficiently annoying to take the extra steps to prevent it from happening.

```
> (doc "<" 'display-form)
(< real1 real2 real3 ...)
(< real1 real2 real3 ...)
```

### Conclusions

This was a fun little project. When I first had the idea, I was really excited because I worked out all of the initial code in less than 2 hours. But, when I started to write this blog post, I started to discover all of the little problems that didn't occur to me initially. Nonetheless, I think that I might have produced something reasonably useful for myself from a modest effort.

***

<a name="1"></a> [1] Scraping code is in a [different repository](https://github.com/hinkelman/chez-docs-scrape) than `chez-docs`.

<a name="2"></a> [2] See this [blog post](/post/getting-started-with-chez-scheme-and-emacs/) for more information on library directories.