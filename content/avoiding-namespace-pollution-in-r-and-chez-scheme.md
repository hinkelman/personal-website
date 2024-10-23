+++
title = "Avoiding namespace pollution in R and Chez Scheme"
date = 2020-02-19
[taxonomies]
tags = ["R", "Chez Scheme"]
+++

I was reading a [blog post](https://white.ucc.asn.au/2020/02/09/whycompositionaljulia.html) that mentioned that Julia has "[w]eak conventions about namespace pollution" and it got me thinking about how I manage namespace pollution in R and Chez Scheme. The short answer is that I don't. I developed bad habits in R centered around writing overly terse code. 

<!-- more -->

These habits emerged in a pre-RStudio, pre-dplyr world where I wasn't using autocomplete and disliked long data frame and function names. For example, I strongly preferred writing 

```
> df$CS <- cumsum(df$Ttl)
``` 

over 

```
> fish_counts$CountTotal_CumSum <- cumsum(fish_counts$CountTotal)
```

My R code was littered with acronyms that made sense when I was writing the code, but were hard to decipher when I came back to it later. Essentially, I was producing [write-only code](https://en.wikipedia.org/wiki/Write-only_language). Because I was usually writing short, one-off scripts that were infrequently revisited, I rarely felt the type of pain that this style can inflict. 

I am a heavy user of 3rd-party packages for R, but my preference for terseness meant that I tended to put all of my `library` calls at the top of the script and re-arrange the order that packages were loaded to manage namespace conflicts [[1]](#1). Honestly, I only today learned that `library` has arguments (`exclude` and `include.only`) for managing the objects that are attached. Because it inevitably leads to more verbose code, I also rarely used `::` (e.g., `dplyr::filter`) to unambigously specify which version of a function that I wanted to use.

I don't profess to have a strong understanding of namespace best practices. Nonetheless, my current understanding is that generally it is a good practice to only load the functions that you are using from a package. In interactive use, though, I don't think that you want to impede your programming flow by constraining the functions immediately available to you. `ggplot2` comes to mind here as a case where you are better off loading the full package functionality. 

When you are using only one function from a package, you might opt for `::` if you only call that function once or twice and `include.only` if you call the function many times in a script. For me, `MESS::auc` is an example where I have typically used the `::` operator because I want the reminder that `auc` is found in the `MESS` packge. Alternatively, I could use `library(MESS, include.only = c("auc"))`.

The approach for importing libraries in Chez Scheme is similar to loading packages in R. To load all the functions in my [`chez-stats` library](https://github.com/hinkelman/chez-stats), you use `(import (chez-stats))`. We can import only the `mean` and `median` procedures with 

```
> (import (only (chez-stats) mean median))
```

We can also reduce the potential for namespace conflicts by importing a library with a prefix.

```
> (import (prefix (chez-stats) stats:))
> (stats:mean '(1 2 2 3 3 3 4 4 4 4))
3
```

My preliminary experience of using the prefix approach greatly improved my autocompletion experience in Emacs. Similarly, a [discussion of function naming conventions in R packages](https://community.rstudio.com/t/function-naming-conventions-and-best-practice/3381) was largely centered on how function prefixes pair nicely with autocompletion in making an R package friendly to new users.

***

<a name="1"></a> [1] The [`conflicted` package](https://conflicted.r-lib.org) offers a stricter way to handle namespace conflicts but I have not tried it.
