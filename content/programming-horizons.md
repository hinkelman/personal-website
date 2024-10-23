+++
title = "Expanding my programming horizons"
date = 2019-02-15
[taxonomies]
tags = ["C++", "Clojure", "Elm", "Julia", "NetLogo", "Pharo", "Python", "R", "Racket", "Red"]
+++

For many years, I've had intentions of learning another programming language. I would guess that I've done 70-80% of my programming work in [R](https://www.r-project.org) and 20-30% in [NetLogo](https://ccl.northwestern.edu/netlogo/). Those two languages have served me well and I haven't yet been in a position where I was required to learn a new language for work. Lately, I've been thinking about my professional development goals and how learning a new programming language might fit into those goals.

<!-- more -->

UPDATE (2020-04-05): I wrote a [retrospective post](/posts/programming-horizons-revisited/) to reflect on what I learned in the year since writing this post.

[Python](https://www.python.org) is the language that has lingered longest on my list of things I should learn. I've poked at Python over the years but never made a serious effort to learn it because I was generally able to find an R-based solution to the problems that I was trying to solve. At various times, I've also thought that [JavaScript](https://www.javascript.com) was a promising choice for me, but then [Shiny](https://shiny.rstudio.com) came along and mostly removed any pressing need to learn JavaScript.

In my current role, arguably, the most obvious professional development choices are to deepen my knowledge of R (still lots to learn even after 10+ years), learn enough C++ (via [Rcpp](http://www.rcpp.org)) to speed up my simulation models, and learn enough JavaScript to extend my Shiny apps. My primary reason for not investing in learning C++ and JavaScript at this time is their reputations as big, messy languages.

Lately, I've spent a lot of time reading about different programming languages. The number of options is overwhelming and I was paralyzed by the thought of making the wrong choice. Then, I came across the following comment on learning programming languages [[1]](#1):

>I’ve dabbled and tinkered in a lot of other languages, always looking for what fits best in my brain, is pleasant to use, and (subjectively) makes me most effective.

This struck me as a much healthier perspective on learning new programming languages. No need to go all-in on a programming language based solely on reading other peoples' opinions. Instead, I should take several languages for a quick test drive and trust my own assessment about whether any of those languages fit me well. Rather than just trusting my gut, though, I decided to identify the features that I thought were most important for my current goals and interests. 

### 1.) Friendly and accessible

I have no computer science background and only domain-specific programming experience. I don't want to struggle with a language that is finicky to get up and running. I want to have access to an IDE that is easy to set up. I've been spoiled by the [RStudio IDE](https://www.rstudio.com/products/rstudio/) and NetLogo's development environment. Mostly, I want to reduce the initial friction to get me to actually start learning the language.

### 2.) GUI capabilities

I really enjoy making the bits and bobs move around on the screen. When I started learning Shiny 5+ yrs ago, I was convinced that web applications were the future. Now, I find myself more interested in learning how to develop desktop applications, preferably without the bloat and baggage of [Electron](https://electronjs.org) (but see my previous [post](/post/deploy-shiny-electron/)).

### 3.) Scientific computing libraries

We are primarily an R shop at Cramer Fish Sciences but I have some latitude to choose a different language for a new project. Languages with existing libraries for scientific computing increase the likelihood that I would be able to use a new language at work. Languages that are faster than R would allow me to more easily [[2]](#2) tackle larger computations (e.g., explore larger parameter space, increase number of replications, etc.).  

### 4.) Broaden my programming experience

I tend to think in an imperative programming style. 'For' loops make frequent appearances in my code. I've not yet strongly embraced the functional features in R or NetLogo. Clearly, there is plenty of room to expand my programming horizons. Because of the apparent benefits of functional programming in data science, I have prioritized learning functional programming concepts over object-oriented programming concepts. But I have the intention to eventually make a serious effort to learn an object-oriented programming language. From an educational standpoint, I'm drawn to learning languages that wholly embrace a single paradigm (e.g., [Elm](https://elm-lang.org), [Smalltalk](https://en.wikipedia.org/wiki/Smalltalk)). For work, though, it seems most pragmatic to focus on languages that are multi-paradigm. I also want to embrace the idea that learning a new language won't necessarily lead to that language replacing R but perhaps allow me to use R more effectively through a deeper understanding of programming in general.

***

With those features in mind, here are my preliminary thoughts on the languages that I've spent the most time reading about (and even writing a few lines of code). 

### [Julia](https://www.julialang.org)

Professionally, Julia is a really obvious choice for me. Julia is designed for numerical and scientific programming with the goal of pairing high-level syntax with performance on par with C. Julia definitely scores well on my scientific computing and performance criteria and I think it fairs reasonably well on the friendly and accessible front. However, I'm not sure if Julia is the obvious choice to expand my programming horizons and my understanding is that it is not yet a very obvious choice for building desktop GUI applications. 

### [Red](https://www.red-lang.org)

When I first learned about Red a few months ago, I was blown away by how easy it makes [GUI programming](https://redprogramming.com/Short%20Red%20Code%20Examples.html). I was also intrigued by the [high priority](https://www.red-lang.org/p/about.html) the Red team places on effective cross-compilation, small executables, and zero dependencies. Red also strives to be a "full-stack" programming language by including a dialect of Red called Red/System for C-level performance. Red scores off the charts on GUI capabilities and meets my criteria of being friendly and approachable. And, through Red/System, performance should be a non-issue. However, Red is arguably a strange choice for scientific computing so my potential uses at work are limited. Mostly, though, Red is in the alpha phase of development and I'm reluctant to invest too much time in it at this point. But I definitely plan to keep an eye on it.

### [Elm](https://elm-lang.org)

Elm is described as a delightful language for reliable web apps. In my limited experience with it, I definitely found it delightful. I was drawn to Elm for its reputation as one of the best languages for learning functional programming. The compiler error messages in Elm are amazing; very helpful to a beginner. Elm scores well on my first and fourth criteria. Obviously, because it is a front-end language, it would also scratch my itch to control the pixels on my screen but it doesn't scratch my growing itch to learn about desktop GUI programming. More obviously, Elm isn't positioned as a tool for scientific computing. I'm not sure if I will swing back around to trying to learn Elm. I guess it depends on my oscillating interest in web development (currently on a down cycle). 

### [Pharo](https://pharo.org)

Pharo is a modern implementation of Smalltalk; a pure, object-oriented programming languge. Smalltalk is lauded for its simple syntax and live-coding environment. It is considered beginner friendly but, honestly, I found it intimidating because it is so different from the type of programming that I've done. In fairness, though, I was just flailing around on my own and not following any tutorials. When I decide to take the plunge and learn object-oriented programming, I will definitely pick up Pharo as my language of choice.

### [Clojure](https://clojure.org)

Clojure is the language on this list that I spent the least time investigating. I had identified Racket (see below) as a prime candidate language to learn early in my process. As I browsed Racket materials, I became concerned that the community was too academic (with emphasis on programming language theory). I eventually identified Clojure as a language similar to Racket but with a community that was more focused on production than research. Arguably, there is a trade-off between my first and third criteria. Emphasizing the 1st criteria favors Racket whereas the 3rd favors Clojure. I decided that the 1st criteria was more important for me at this stage and re-upped on my commitment to learn Racket.

### [Racket](https://www.racket-lang.org)

Racket managed to stay at the top of my list of "next programming languages to learn" despite my flirtation with several other languages. Racket started as a Scheme implementation but has grown to include the "best of Scheme and Lisp." The killer feature of Racket is being able to easily implement your own programming languages in Racket. Honestly, that is not a feature that stirs much interest for me but maybe I will grow to appreciate it later. Mostly, I was drawn to Racket because it has a reputation as beginner friendly with a good IDE ([DrRacket](https://docs.racket-lang.org/drracket/index.html)). It is dead simple to install Racket and get started with DrRacket. On my short list here, only Pharo is comparable. The Scheme heritage means that Racket has simple syntax. Thus, Racket fully meets my first criteria. Racket is a general-purpose language that comes "batteries included" with an extensive standard library, including a GUI toolkit, which ticks my second box. On the face of it, Racket is a reasonable choice for scientific computing but [has not been widely embraced in that domain](https://github.com/racket/racket/wiki/Scientific-Computing). Nonetheless, Racket has decent performance and generally outperforms Python in the [benchmarks game.](https://benchmarksgame-team.pages.debian.net/benchmarksgame/fastest/racket-python3.html) I'm not sure if Racket will expand my programming horizons as much as languages like Elm and Pharo but I expect the ways that it expands my programming experience to be highly relevant to my work in R because both Racket and R have a Scheme and Lisp heritage. 

***

<a name="1"></a> [1] [Quora answer by Gregg Irwin](https://www.quora.com/Why-is-using-a-GUI-in-most-of-all-programming-languages-such-a-hassle-given-that-Rebol-and-Red-have-such-elegant-solutions)

<a name="2"></a> [2] By more easily, I'm thinking about languages that might have similar expressiveness to R but better performance without dropping down to C/C++.

