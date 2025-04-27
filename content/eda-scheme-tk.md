+++
title = "Exploratory data analysis with Scheme, Gnuplot, and Tk"
date = 2024-12-26
[taxonomies]
tags = ["Scheme", "Chez Scheme", "dataframe", "gnuplot-pipe", "chez-tk"]
+++

In my [second post on this blog](/programming-horizons/), I expressed an interest in learning how to build desktop applications. I have yet to pursue that interest. Instead, I've primarily continued developing [Shiny](https://shiny.posit.co) apps deployed on the web (but see [Shiny Scorekeeper](https://github.com/hinkelman/Shiny-Scorekeeper)). Recently, though, I've spent some time learning about the [Tk GUI toolkit](https://www.tcl.tk) for developing desktop applications. In this post, I revisit an old [post](/eda-scheme/) using the [`dataframe`](https://github.com/hinkelman/dataframe/) and [`gnuplot-pipe`](https://github.com/hinkelman/gnuplot-pipe) libraries for Scheme to conduct simple exploratory data analysis (EDA) and add an interface with [`chez-tk`](https://github.com/hinkelman/chez-tk/).

<!-- more -->

## Tcl/Tk

A few months ago I saw on [Hacker News](https://news.ycombinator.com/item?id=41661906) that Tcl/Tk accounced a major release. My interest was piqued by the generally favorable comments and comparisons to Lisp. I also like the generality that comes from the large number of languages that provide bindings to Tk. One of those languages is my primary programming language, R. It seems that the GUI options for R have largely collapsed to only Shiny and [`tcltk`](https://r-universe.dev/manuals/tcltk.html). To satisfy my curiousity, I tackled a little [project](https://github.com/hinkelman/ZoopSynth-tcltk) to learn more about the `tcltk` package in R.

I was also interested in available bindings to Tk for my favorite programming language, Scheme, and found [PS/Tk](https://snow-fort.org/s/peterlane.info/peter/rebottled/pstk/1.7.0/index.html). 

> The PSTK library has had a long history in the Scheme community and, in one form or another, is available for many Scheme implementations. The current file includes its history starting from an implementation of Chicken/Tk by Wolf-Dieter Busch from 2004 based on earlier code by Sven Hartrumpf from 1997. Nils Holm made the library portable, and so created PSTK. Ken Dickey created an R6RS version.

PS/Tk communicates with Tcl/Tk through a process port. The versions of PS/Tk code that I found did not include R6RS compatibility so I added the code needed to open a process port in Chez Scheme, converted from R7RS to R6RS library, called it `chez-tk`, and submitted it to [Akku](https://akkuscm.org/packages/chez-tk/). I've collected [examples](https://github.com/hinkelman/chez-tk/tree/main/examples) for using `chez-tk` based on existing PS/Tk examples or translating examples from [TkDocs](https://tkdocs.com/). In translating those examples, I was primarily using [`tkinter`](https://docs.python.org/3/library/tkinter.html) for Python to understand how they work. I was impressed by the autocompletion and documentation for `tkinter` available through VS Code. For `tcltk` and `chez-tk`, you have to learn the translation rules and use the Tcl/Tk documentation.

After working up a set of `chez-tk` examples, the only apparent bug that I found was related to the [inclusion of parentheses in listbox choices](https://github.com/hinkelman/chez-tk/issues/1). It was a small change in the code to fix that bug, and I didn't notice any negative impacts when running through the examples after making the change, but I also don't understand why that procedure was written that way in the first place. That's an uneasy feeling.

## App Overview

When I first started making Shiny apps 10+ years ago, I was drawn to the potential to make my work in R accessible to non-R users. I didn't anticipate the extent to which I would find graphical user interfaces to multi-dimensional datasets useful for my own data exploration efforts. This `chez-tk` example is made in that spirit, i.e., the app isn't packaged into a standalone desktop application for other users. The user needs to know how to use Chez Scheme and Akku and needs to install Tcl/Tk and Gnuplot. 

The app allows for filtering on years, months, and cities and summarizes an annual or monthly time series grouped by city for the selected response variable. Each click of the button will generate a plot in a new window. 

![](/img/EDA-App-Screenshot.png)

## Data Preparation

### Libraries

All of the libraries in the import statement below are available through [Akku](https://akkuscm.org/packages/). `dataframe` is used for data manipulation. Only one procedure is imported from `wax irregex` to process a string that is returned from Tcl/Tk. `gnuplot-pipe` is used for plotting. `chez-tk` allows us to build a user interface. [All of the code is available in a single file in this [gist](https://gist.github.com/hinkelman/7236f4041dc56ed15541e3cad65d7626).]

```
(import (dataframe)
        (only (wak irregex) irregex-split)
        (prefix (gnuplot-pipe) gp:)
        (chez-tk))
```

### Data

We are using the Texas housing dataset included as part of the [`ggplot2`](https://ggplot2.tidyverse.org/) package for R. I've written that dataset to a [CSV file](/data/txhousing.csv) for use in this post.

```
> (define df (csv->dataframe "txhousing.csv"))

> (dataframe-display df)

 dim: 8602 rows x 9 cols
     city    year   month   sales    volume  median  listings  inventory       date 
    <str>   <num>   <num>   <num>     <num>   <num>     <num>      <num>      <num> 
  Abilene   2000.      1.     72.  5.380E+6  71400.      701.     6.3000  2000.0000 
  Abilene   2000.      2.     98.  6.505E+6  58700.      746.     6.6000  2000.0833 
  Abilene   2000.      3.    130.  9.285E+6  58100.      784.     6.8000  2000.1667 
  Abilene   2000.      4.     98.  9.730E+6  68600.      785.     6.9000  2000.2500 
  Abilene   2000.      5.    141.  1.059E+7  67300.      794.     6.8000  2000.3333 
  Abilene   2000.      6.    156.  1.391E+7  66900.      780.     6.6000  2000.4167 
  Abilene   2000.      7.    152.  1.264E+7  73500.      742.     6.2000  2000.5000 
  Abilene   2000.      8.    131.  1.071E+7  75000.      765.     6.4000  2000.5833 
  Abilene   2000.      9.    104.  7.615E+6  64500.      771.     6.5000  2000.6667 
  Abilene   2000.     10.    101.  7.040E+6  59300.      764.     6.6000  2000.7500 
```

### Global Variables

We define global variables based on `df` for use in the app. First, though, we need to define a helper procedure to double quote any strings in a list that have spaces because `chez-tk` appends those strings into one big string for passing to Tcl/Tk. Ideally, `chez-tk` would handle this for us, but, for now, I'm reluctant to make too many changes to `chez-tk` (see above).

We get the list of cities from the dataframe and remove duplicates. For some reason, there is a problem with three of the cities causing the app to crash with a message about "invalid listvar values." I have no idea why just those three cities cause a problem, but, for now, I've decided not to try to chase down that problem. The other thing to point out is that the options for the combobox are defined as a single string in `vars-labs`, which also shows the double quoting requirement. When a response variable is selected in the app, the column name is looked up using the associations in `vars`. 

```
(define (double-quote lst)
  (map (lambda (x)
	 (let ([x-list (string->list x)])
	   (if (member #\space x-list)
	       (string-append "\"" x "\"")
	       x)))
       lst))

(define cities (remove-duplicates ($ df 'city)))
(define cities
  (filter
   (lambda (x) (not (member x '("Montgomery County"
                                "Port Arthur"
                                "Wichita Falls"))))
   cities))
(define cities-dq (double-quote cities))

(define min-yr (apply min ($ df 'year)))
(define max-yr (apply max ($ df 'year)))

(define months '(Jan Feb Mar Apr May Jun
		     Jul Aug Sep Oct Nov Dec))

(define vars '(("Median Sale Price" median)
	       ("Sales" sales)
	       ("Volume" volume)
	       ("Listings" listings)
	       ("Inventory" inventory)))

(define vars-labs "\"Median Sale Price\" Sales Volume Listings Inventory")
```

### Filter

We create a small wrapper procedure around a standard `dataframe-filter*`. The `month` column in `df` is represented by numeric months. If we add one to the indices of the selected months, then we get the numeric months for use in the filter. We use the cities indices to get the city names from `cities` (not `cities-dq`). We remove all rows with missing values in `resp-var` with `dataframe-remove-na`. If there were missing values in `year`, `month`, and `city` columns, then we would need to add them to the `dataframe-remove-na`.

```
(define (filter-data df min-yr max-yr months-idx cities-idx cities resp-var)
  (let ([months-sel (map add1 months-idx)]
        [cities-sel (map (lambda (x) (list-ref cities x)) cities-idx)])
    (-> df
        (dataframe-remove-na resp-var)
        (dataframe-filter*
         (city year month)
         (and (>= year min-yr)
	      (<= year max-yr)
	      (member month months-sel)
	      (member city cities-sel))))))
```

### Aggregate

As with filtering, we are wrapping `dataframe-aggregate`. In this case, though, we can't use the macro version (indicated with a trailing `*`). The macro versions of the dataframe verbs are intended for interactive use and provide simpler syntax. The grouping variables are `city` and `xvar`. The new column is named `mean-rv` where 'rv' stands for response variable. `(list (list resp-var))` provides the names of the columns used in the lambda expressions (one sub-list for each expression).

```
(define (agg-data df xvar resp-var)
  (dataframe-aggregate
   df
   (list 'city xvar)
   '(mean-rv)
   (list (list resp-var))
   (lambda (resp-var) (exact->inexact (mean resp-var)))))
```

### Plot

`gp:send` sends commands to Gnuplot as strings. When setting the axis labels, we need to surround the label with single quotation marks to distinguish the label from the rest of the command string. To plot multiple sets of data, `gp:plot` accepts a list where the first item is a string with optional properties (e.g., title provides a label for the legend), the second is a list with x-coordinates, and the third is a list with y-coordinates.

```
(define (plot-data df x y xvar-str resp-var-str)
  (gp:call/gnuplot
   (gp:send "set key top left")
   (gp:send (string-append "set xlabel \'" xvar-str "\'"))
   (gp:send (string-append "set ylabel \'Avg. " resp-var-str "\'"))
   (gp:send "set style data linespoints")
   (gp:plot
    (map (lambda (c)
           (let ([df-sub (dataframe-filter*
                          df
                          (city)
                          (string=? c city))])
             (list
              (string-append "title '" c "'")
              ($ df-sub x)
              ($ df-sub y))))
         (remove-duplicates ($ df 'city))))))
```

## App Details

### Named Frames and Widgets

[Tile](https://tktable.sourceforge.net/tile/) provides reimplementations of many classic widgets in the `ttk` namespace. In the first line in the code block below, we opt to use the Tile versions over the classic versions for all widgets with `ttk-map-widgets`. `tk-start` initializes the Tk shell.

In `chez-tk`, widgets are represented as procedures that can be used to configure the widget. In this app, we use frames, labels, spinboxes, listboxes, radiobuttons, a combobox, and a button, but only a few of them require names for subsequent configuration. The named procedures also specify the relationship of the frames and widgets, e.g., `tk` creates `frame` and `frame` creates `months-lb`, `cities-lb`, `vars-cb`, and all other widgets.

Commands are represented as symbols (e.g., `'create-widget`) whereas parameters are represented as symbols with trailing colons (e.g., `'height:`). Scheme symbols can be used in place of strings and Scheme values such as `#f` are converted to the Tcl/Tk equivalent. `tk-var` associates a Tk variable name with a widget. 

When `'exportselection:` is set to `#t`, clicking outside of the listbox deselects any listbox selections. For multiple selections in listboxes, the `'selectmode:` needs to be set to `'multiple` or `'extended`. 

> If the selection mode is multiple or extended, any number of elements may be selected at once, including discontiguous ranges. In multiple mode, clicking button 1 on an element toggles its selection state without affecting any other elements. In extended mode, pressing button 1 on an element selects it, deselects everything else, and sets the anchor to the element under the mouse; dragging the mouse with button 1 down extends the selection to include all the elements between the anchor and the element under the mouse, inclusive.

```
(ttk-map-widgets 'all)
(define tk (tk-start))
(define frame (tk 'create-widget 'frame 'padding: '(10 10 10 10)))
(define months-lb
  (frame 'create-widget 'listbox 'listvariable: (tk-var 'months-tk)
	 'height: 5 'exportselection: #f 'selectmode: 'extended))
(define cities-lb
  (frame 'create-widget 'listbox 'listvariable: (tk-var 'cities-tk)
	 'height: 10 'exportselection: #f 'selectmode: 'extended))
(define vars-cb
  (frame 'create-widget 'combobox 'values: vars-labs 'state: 'readonly))
```

### App Layout

We are using the grid geometry manager with a simple layout of three columns and nine rows all contained in a single frame. Widgets are sized to the content so a long label like `Response Variable` should be set to span multiple columns to prevent undesirable extra space. The `'sticky:` parameter uses cardinal directions (`'nwes`) for alignment of widgets. Most widgets can be created within a call to `tk/grid` because there is no subsequent configuration of those widgets, just setting and getting the Tk variable.

```
(tk/grid frame)
(tk/grid (frame 'create-widget 'label 'text: "Years")
	 'column: 0 'row: 0 'sticky: 'w 'pady: 5)
(tk/grid (frame 'create-widget 'spinbox 'from: min-yr 'to: max-yr
		'textvariable: (tk-var 'min-yr-tk) 'width: 5)
	 'column: 1 'row: 0 'sticky: 'w)
(tk/grid (frame 'create-widget 'spinbox 'from: min-yr 'to: max-yr
		'textvariable: (tk-var 'max-yr-tk) 'width: 5)
	 'column: 2  'row: 0 'sticky: 'w)

(tk/grid (frame 'create-widget 'label 'text: "Months")
	 'column: 0 'row: 1 'sticky: 'w)
(tk/grid months-lb 'column: 0 'row: 2 'columnspan: 3 'sticky: 'we 'pady: 5)

(tk/grid (frame 'create-widget 'label 'text: "Cities")
	 'column: 0 'row: 3 'sticky: 'w)
(tk/grid cities-lb 'column: 0 'row: 4 'columnspan: 3 'sticky: 'we 'pady: 5)

(tk/grid (frame 'create-widget 'label 'text: "X Variable")
	 'column: 0 'row: 5 'sticky: 'w 'pady: 5)
(tk/grid (frame 'create-widget 'radiobutton 'text: "Year" 'value: "Year"
	        'variable: (tk-var 'xvar-tk))
	 'column: 1 'row: 5 'sticky: 'e)
(tk/grid (frame 'create-widget 'radiobutton 'text: "Month" 'value: "Month"
	        'variable: (tk-var 'xvar-tk))
	 'column: 2 'row: 5 'sticky: 'e)

(tk/grid (frame 'create-widget 'label 'text: "Response Variable")
	 'column: 0 'row: 6 'columnspan: 3 'sticky: 'w)
(tk/grid vars-cb 'column: 0 'row: 7 'columnspan: 3 'sticky: 'we)
```

### Commands

In this app, we only have one command, `plot-cmd`, which filters, aggregates, and plots data. `plot-cmd` is associated with the plot button via the `'command:` parameter. A command procedure takes no arguments. Within `plot-cmd`, we retrieve the state of all of the app widgets at the time that the button was clicked and then pass those values to the procedures that we described above, i.e., `filter-data`, `agg-data`, and `plot-data`.

For several widgets, we use `tk-get-var` with the Tk variable name to get the current widget value and convert it to the appropriate type. For the combobox, we use the Scheme name (`vars-cb`) with `'get`. Similarly, for listboxes, we use the Scheme name with `'curselection`, which returns a string of the selected indices, e.g., `"0 3 4 8"`. `prepare-curselection` splits that string into a list of numeric indices for use in `filter-data`.

If the filtering and aggregating steps produce an empty dataframe, then clicking on the plot button has no effect (because otherwise the app would crash). Ideally, the user would receive feedback on why the plot isn't displayed, but, unfortunately, that feature is not a simple addition to the app (based on my current understanding of Tcl/Tk).

```
(define (prepare-curselection x)
  (map string->number (irregex-split " " x)))

(define plot-cmd
  (lambda ()
    (let* ([xvar-str (tk-get-var 'xvar-tk)]
           [xvar (if (string=? xvar-str "Year") 'year 'month)]
           [rv-str (vars-cb 'get)]
           [rv (cadr (assoc rv-str vars))]
           [df-sub (filter-data
                    df
                    (string->number (tk-get-var 'min-yr-tk))
                    (string->number (tk-get-var 'max-yr-tk))
                    (prepare-curselection (months-lb 'curselection))
                    (prepare-curselection (cities-lb 'curselection))
                    cities
                    rv)])
      ;; can't aggregate empty dataframe
      (when (> (car (dataframe-dim df-sub)) 0)
        (plot-data (agg-data df-sub xvar rv) xvar 'mean-rv xvar-str rv-str)))))

(tk/grid (frame 'create-widget 'button 'text: "Plot" 'command: plot-cmd)
	 'column: 0 'row: 8 'columnspan: 3 'sticky: 'we 'pady: 5)
```

### Initial Values

For spinboxes and radiobuttons, `tk-set-var!` sets the initial value. For the combobox, the initial value is set with the Scheme name, `vars-cb`, and `'set`. For listboxes, `tk-set-var!` sets the options, but the initial values are set with the Scheme name. Here is the Tcl/Tk documentation for listbox selection set:

> pathName selection set first ?last?  
Selects all of the elements in the range between first and last, inclusive, without affecting the selection state of elements outside that range.

This nicely illustrates the translation between Tk and `chez-tk` where the Scheme name is used in place of the pathName and the rest of the expression is almost identical. For `months-lb`, we initially select all months. For `cities-lb`, we are selecting multiple elements that are not part of an inclusive range so we set the selected values iteratively. We use a helper procedure, `get-idx`, to get indices for a subset of the cities. 

```
(tk-set-var! 'min-yr-tk min-yr)
(tk-set-var! 'max-yr-tk max-yr)
(tk-set-var! 'months-tk months)
(tk-set-var! 'cities-tk cities-dq)
(tk-set-var! 'xvar-tk "Year")
(vars-cb 'set "Median Sale Price")
(months-lb 'selection 'set 0 11)

(define (get-idx lst lst-sub)
  ;; get indices of lst-sub from lst
  (let* ([idx (iota (length lst))]
	 [lst-idx (map (lambda (x i) (cons x i)) lst idx)])
    (map (lambda (y) (cdr (assoc y lst-idx))) lst-sub)))

(for-each (lambda (x) (cities-lb 'selection 'set x))
	  (get-idx cities '("Austin" "Dallas" "El Paso" "Houston"
                            "Lubbock" "San Antonio")))
```

## Conclusions

I have enjoyed learning the basics of making GUIs with Tk in Scheme and R. I don't mind the outdated look of the widgets and I like the compactness of the interface (compared to a Shiny app). I think it would be fun to make a `chez-tk` version of my [Shiny-Scorekeeper app](https://github.com/hinkelman/Shiny-Scorekeeper) (or even a `tcltk` version in R). I'm also interested in the possibility of packaging `chez-tk` and `tcltk` apps into standalone executables.