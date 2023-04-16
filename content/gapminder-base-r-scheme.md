+++
title = "Analyzing gapminder dataset with base R and Scheme"
date = 2021-04-30
updated = 2023-04-15
[taxonomies]
categories = ["Chez Scheme", "dataframe", "chez-stats"]
tags = ["dataframe", "dplyr", "pandas", "EDA", "filter", "modify", "aggregate"]
+++

I keep my eye out for blog posts illustrating data analysis tasks in R that I can use to test the functionality of my [`chez-stats`](https://github.com/hinkelman/chez-stats) and [`dataframe`](https://github.com/hinkelman/dataframe/) libraries for Chez Scheme [[1]](#1). A [post](https://appsilon.com/pandas-vs-dplyr/) comparing [`pandas`](https://pandas.pydata.org/) (Python) and [`dplyr`](https://dplyr.tidyverse.org/) (R) in a basic analysis of the gapminder dataset provides a nice little test case. In this post, I will also include base R code used to accomplish the same tasks as a contrast to both the Scheme code and the `dplyr` code from the other post. 

<!-- more -->

## Data Loading

The R package, [`gapminder`](https://cran.r-project.org/web/packages/gapminder/), provides an excerpt of the data available at [Gapminder.org](https://www.gapminder.org/). I've written the data from that package to a CSV file ([available here](/data/gapminder.csv)).

*Base R*

I've added a little function, `head10`, to simplify subsequent code. It is not necessary. By default, `head` provides six rows. I've changed it to 10 to match the default for `dataframe-display`. The main thing to note here is that because we are using base R, we don't need to load any packages.

```
> gapminder <- read.csv("gapminder.csv")
> head10 <- function(data) head(data, n = 10)
> head10(gapminder)

       country continent year lifeExp      pop gdpPercap
1  Afghanistan      Asia 1952  28.801  8425333  779.4453
2  Afghanistan      Asia 1957  30.332  9240934  820.8530
3  Afghanistan      Asia 1962  31.997 10267083  853.1007
4  Afghanistan      Asia 1967  34.020 11537966  836.1971
5  Afghanistan      Asia 1972  36.088 13079460  739.9811
6  Afghanistan      Asia 1977  38.438 14880372  786.1134
7  Afghanistan      Asia 1982  39.854 12881816  978.0114
8  Afghanistan      Asia 1987  40.822 13867957  852.3959
9  Afghanistan      Asia 1992  41.674 16317921  649.3414
10 Afghanistan      Asia 1997  41.763 22227415  635.3414
```

*Chez Scheme*

The code here is more verbose than in base R. We need to import a couple of libraries. Also, `read-delim` reads data row-wise, which I'm calling a `rowtable`, and needs to be converted to a column-wise `dataframe`. Moreover, `read-delim` doesn't do any type conversion; all values are read as strings. In this example, 4 of 6 columns need to be converted from `string` to `number`. 

```
> (import (chez-stats)
          (dataframe))
> (define gapminder
    (-> (read-delim "gapminder.csv")
        (rowtable->dataframe #t)
        (dataframe-modify-at
         string->number 'year 'lifeExp 'pop 'gdpPercap)))
> (dataframe-display gapminder)

 dim: 1704 rows x 6 cols
      country  continent   year  lifeExp       pop  gdpPercap 
  Afghanistan       Asia  1952.  28.8010  8.425E+6   779.4453 
  Afghanistan       Asia  1957.  30.3320  9.241E+6   820.8530 
  Afghanistan       Asia  1962.  31.9970  1.027E+7   853.1007 
  Afghanistan       Asia  1967.  34.0200  1.154E+7   836.1971 
  Afghanistan       Asia  1972.  36.0880  1.308E+7   739.9811 
  Afghanistan       Asia  1977.  38.4380  1.488E+7   786.1134 
  Afghanistan       Asia  1982.  39.8540  1.288E+7   978.0114 
  Afghanistan       Asia  1987.  40.8220  1.387E+7   852.3959 
  Afghanistan       Asia  1992.  41.6740  1.632E+7   649.3414 
  Afghanistan       Asia  1997.  41.7630  2.223E+7   635.3414 
```

## Filtering

### Problem 1

Filter the dataset to retain only rows where `year` is `2007`.

*Base R*

```
> head10(gapminder[gapminder$year == 2007, ])

        country continent year lifeExp       pop  gdpPercap
12  Afghanistan      Asia 2007  43.828  31889923   974.5803
24      Albania    Europe 2007  76.423   3600523  5937.0295
36      Algeria    Africa 2007  72.301  33333216  6223.3675
48       Angola    Africa 2007  42.731  12420476  4797.2313
60    Argentina  Americas 2007  75.320  40301927 12779.3796
72    Australia   Oceania 2007  81.235  20434176 34435.3674
84      Austria    Europe 2007  79.829   8199783 36126.4927
96      Bahrain      Asia 2007  75.635    708573 29796.0483
108  Bangladesh      Asia 2007  64.062 150448339  1391.2538
120     Belgium    Europe 2007  79.441  10392226 33692.6051
```

*Chez Scheme*

This example introduces the thread-first operator (`->`), which takes the result of the previous procedure and passes it to the first argument of the next procedure. For data analysis, I strongly prefer the threading (or piping) approach over writing the code inside-out or creating lots of intermediate datasets.

```
> (-> gapminder
      (dataframe-filter
       (filter-expr (year) (= year 2007)))
      dataframe-display)

 dim: 142 rows x 6 cols
      country  continent   year  lifeExp       pop  gdpPercap 
  Afghanistan       Asia  2007.  43.8280  3.189E+7   974.5803 
      Albania     Europe  2007.  76.4230  3.601E+6  5937.0295 
      Algeria     Africa  2007.  72.3010  3.333E+7  6223.3675 
       Angola     Africa  2007.  42.7310  1.242E+7  4797.2313 
    Argentina   Americas  2007.  75.3200  4.030E+7 12779.3796 
    Australia    Oceania  2007.  81.2350  2.043E+7 34435.3674 
      Austria     Europe  2007.  79.8290  8.200E+6 36126.4927 
      Bahrain       Asia  2007.  75.6350  7.086E+5 29796.0483 
   Bangladesh       Asia  2007.  64.0620  1.504E+8  1391.2538 
      Belgium     Europe  2007.  79.4410  1.039E+7 33692.6051 
```

### Problem 2

Filter the dataset to retain only rows where `year` is `2007` and `continent` is `Americas`.

*Base R*

In this case, I've used `subset` rather than the more conventional `[` subsetting used in Problem 1. The advantage of `subset` is that I only need to type `gapminder` once instead of three times.

```
> head10(subset(gapminder, year == 2007 & continent == "Americas"))

               country continent year lifeExp       pop gdpPercap
60           Argentina  Americas 2007  75.320  40301927 12779.380
144            Bolivia  Americas 2007  65.554   9119152  3822.137
180             Brazil  Americas 2007  72.390 190010647  9065.801
252             Canada  Americas 2007  80.653  33390141 36319.235
288              Chile  Americas 2007  78.553  16284741 13171.639
312           Colombia  Americas 2007  72.889  44227550  7006.580
360         Costa Rica  Americas 2007  78.782   4133884  9645.061
396               Cuba  Americas 2007  78.273  11416987  8948.103
444 Dominican Republic  Americas 2007  72.235   9319622  6025.375
456            Ecuador  Americas 2007  74.994  13755680  6873.262
```

*Chez Scheme*

`dataframe-filter` was designed to avoid having to repeat the dataframe name in each sub-expression of the `filter-expr`, but requires typing `year` and `continent` two times each in this example.

```
> (-> gapminder
      (dataframe-filter
       (filter-expr (year continent)
                    (and (= year 2007)
                         (string=? continent "Americas"))))
      dataframe-display)

 dim: 25 rows x 6 cols
             country  continent   year  lifeExp       pop  gdpPercap 
           Argentina   Americas  2007.  75.3200  4.030E+7 12779.3796 
             Bolivia   Americas  2007.  65.5540  9.119E+6  3822.1371 
              Brazil   Americas  2007.  72.3900  1.900E+8  9065.8008 
              Canada   Americas  2007.  80.6530  3.339E+7 36319.2350 
               Chile   Americas  2007.  78.5530  1.628E+7 13171.6389 
            Colombia   Americas  2007.  72.8890  4.423E+7  7006.5804 
          Costa Rica   Americas  2007.  78.7820  4.134E+6  9645.0614 
                Cuba   Americas  2007.  78.2730  1.142E+7  8948.1029 
  Dominican Republic   Americas  2007.  72.2350  9.320E+6  6025.3748 
             Ecuador   Americas  2007.  74.9940  1.376E+7  6873.2623 
```

### Problem 3

Filter the dataset to retain only rows where `year` is `2007` and `continent` is `Americas` and `country` is `United States`. This last filter is a little silly because the `country` condition makes the `continent` condition unnecessary, but the point is to show how code complexity increases with additional conditions.

*Base R*

Thanks to `subset`, adding additional conditions is straightforward with zero redundancy.

```
> subset(gapminder, 
         year == 2007 & 
           continent == "Americas" &
           country == "United States")

           country continent year lifeExp       pop gdpPercap
1620 United States  Americas 2007  78.242 301139947  42951.65
```

*Chez Scheme*

```
> (-> gapminder
      (dataframe-filter
       (filter-expr (year continent country)
                    (and (= year 2007)
                         (string=? continent "Americas")
                         (string=? country "United States"))))
      dataframe-display)

 dim: 1 rows x 6 cols
        country  continent   year  lifeExp       pop  gdpPercap 
  United States   Americas  2007.  78.2420  3.011E+8   42951.65 
```

## Summary Statistics

### Problem 1

Calculate the average life expectancy worldwide in 2007.

*Base R*

```
> mean(subset(gapminder, year == 2007)$lifeExp)
[1] 67.00742
```

*Chez Scheme*

This example introduces the `$` operator, which was inspired by R, to extract the values of a column (e.g., `lifeExp`). The difference in verbosity between base R and Chez Scheme in this example is related to the filtering step. Extracting a column and calculating the mean involve nearly the same number of characters. `mean` is provided by `chez-stats`, but is simple procedure. `(mean x)` is equivalent to `(/ (apply + x) (length x))`.

```
> (-> gapminder
      (dataframe-filter
       (filter-expr (year) (= year 2007)))
      ($ 'lifeExp)
      (mean))
67.00742253521126    
```

### Problem 2

Calculate the average life expectancy for every continent in 2007.

*Base R*

`aggregate` allows for use of the formula syntax (e.g., `lifeExp ~ continent`) to concisely describe the summarized value (`lifeExp`) and grouping variable(s).

```
> aggregate(lifeExp ~ continent, 
           data = subset(gapminder, year == 2007),
           FUN = mean)

  continent  lifeExp
1    Africa 54.80604
2  Americas 73.60812
3      Asia 70.72848
4    Europe 77.64860
5   Oceania 80.71950
```

*Chez Scheme*

One difference to note here is that `dataframe-aggregate` doesn't automatically sort by the grouping variable. We would have to explicitly add that sorting step.

```
> (-> gapminder
      (dataframe-filter
       (filter-expr (year) (= year 2007)))
      (dataframe-aggregate
       '(continent)
       (aggregate-expr (mean-lifeExp (lifeExp) (mean lifeExp))))
      (dataframe-display))

 dim: 5 rows x 2 cols
  continent  mean-lifeExp 
    Oceania       80.7195 
     Europe       77.6486 
   Americas       73.6081 
       Asia       70.7285 
     Africa       54.8060 
```

### Problem 3

Calculate the total population per continent in 2007 and sort the results in descending order of total population.

*Base R*

```
> total_pop = aggregate(pop ~ continent, 
                        data = subset(gapminder, year == 2007),
                        FUN = sum)
> total_pop[order(-total_pop$pop),]

  continent        pop
3      Asia 3811953827
1    Africa  929539692
2  Americas  898871184
4    Europe  586098529
5   Oceania   24549947
```

*Chez Scheme*

```
> (-> gapminder
      (dataframe-filter
       (filter-expr (year) (= year 2007)))
      (dataframe-aggregate
       '(continent)
       (aggregate-expr (total-pop (pop) (apply + pop))))
      (dataframe-sort (sort-expr (> total-pop)))
      (dataframe-display))

 dim: 5 rows x 2 cols
  continent  total-pop 
       Asia   3.812E+9 
     Africa   9.295E+8 
   Americas   8.989E+8 
     Europe   5.861E+8 
    Oceania   2.455E+7 
```

## Creating Derived Columns

### Problem 1

Calculate the total GDP by multiplying `pop` and `gdpPercap`.

*Base R*

The help page for `transform` advises that it is only intended for interactive use. Alternatively, you could use: `gapminder$GPD = gapminder$pop * gapminder$gdpPercap`. 

```
> head10(transform(gapminder, GDP = pop * gdpPercap))

       country continent year lifeExp      pop gdpPercap         GDP
1  Afghanistan      Asia 1952  28.801  8425333  779.4453  6567086330
2  Afghanistan      Asia 1957  30.332  9240934  820.8530  7585448670
3  Afghanistan      Asia 1962  31.997 10267083  853.1007  8758855797
4  Afghanistan      Asia 1967  34.020 11537966  836.1971  9648014150
5  Afghanistan      Asia 1972  36.088 13079460  739.9811  9678553274
6  Afghanistan      Asia 1977  38.438 14880372  786.1134 11697659231
7  Afghanistan      Asia 1982  39.854 12881816  978.0114 12598563401
8  Afghanistan      Asia 1987  40.822 13867957  852.3959 11820990309
9  Afghanistan      Asia 1992  41.674 16317921  649.3414 10595901589
10 Afghanistan      Asia 1997  41.763 22227415  635.3414 14121995875
```

*Chez Scheme*

```
> (-> gapminder
      (dataframe-modify
       (modify-expr (GDP (pop gdpPercap) (* pop gdpPercap))))
      (dataframe-display))

     dim: 1704 rows x 7 cols
      country  continent   year  lifeExp       pop  gdpPercap        GDP 
  Afghanistan       Asia  1952.  28.8010  8.425E+6   779.4453  6.567E+09 
  Afghanistan       Asia  1957.  30.3320  9.241E+6   820.8530  7.585E+09 
  Afghanistan       Asia  1962.  31.9970  1.027E+7   853.1007  8.759E+09 
  Afghanistan       Asia  1967.  34.0200  1.154E+7   836.1971  9.648E+09 
  Afghanistan       Asia  1972.  36.0880  1.308E+7   739.9811  9.679E+09 
  Afghanistan       Asia  1977.  38.4380  1.488E+7   786.1134  1.170E+10 
  Afghanistan       Asia  1982.  39.8540  1.288E+7   978.0114  1.260E+10 
  Afghanistan       Asia  1987.  40.8220  1.387E+7   852.3959  1.182E+10 
  Afghanistan       Asia  1992.  41.6740  1.632E+7   649.3414  1.060E+10 
  Afghanistan       Asia  1997.  41.7630  2.223E+7   635.3414  1.412E+10 
```

### Problem 2

Find the top 10 countries in percentile of `gdpPercap`.

*Base R*

```
> percentile <- function(x){
    rank_x = rank(x)
    rank_x/max(rank_x)
  }

> gapminder2007 = transform(subset(gapminder, year == 2007),
                            percentile = percentile(gdpPercap))
> head10(gapminder2007[order(-gapminder2007$percentile),])

              country continent year lifeExp       pop gdpPercap percentile
1152           Norway    Europe 2007  80.196   4627926  49357.19  1.0000000
864            Kuwait      Asia 2007  77.588   2505559  47306.99  0.9929577
1368        Singapore      Asia 2007  79.972   4553009  47143.18  0.9859155
1620    United States  Americas 2007  78.242 301139947  42951.65  0.9788732
756           Ireland    Europe 2007  78.885   4109086  40676.00  0.9718310
672  Hong Kong, China      Asia 2007  82.208   6980412  39724.98  0.9647887
1488      Switzerland    Europe 2007  81.701   7554661  37506.42  0.9577465
1092      Netherlands    Europe 2007  79.762  16570613  36797.93  0.9507042
252            Canada  Americas 2007  80.653  33390141  36319.24  0.9436620
696           Iceland    Europe 2007  81.757    301931  36180.79  0.9366197
```

*Chez Scheme*

This example reveals a weak spot in the `dataframe` API. The issue is that `rank%` operates on a `list` whereas `dataframe-modify` expects that the `modify-expr` takes only scalars as inputs because the `modify-expr` is mapped over all rows in a `dataframe`. The workaround to avoid the mapping is to specify that no columns (i.e., `()`) from the dataframe are used in the `modify-expr`, then the list created in the subsequent expression (e.g., `(rank% ($ df 'gdpPercap))`) will be used as the column values.
 
 Because the `dataframe` library only has thread-first (`->`) and thread-last (`->>`) operators, we have to create the awkward `lambda` procedure to get the output of `dataframe-filter` to the right place in `dataframe-modify`. [SRFI 197](https://srfi.schemers.org/srfi-197/srfi-197.html) provides a pipeline operator that requires that the location of the output of the previous procedure is explicitly specified with `_`, which would likely simplify this code. An earlier, simpler version of SRFI 197 was the source for `->` and `->>` in the `dataframe` library.

```
> (define (rank% lst)
    (let* ([rank-lst (rank lst 'mean)]
           [max-rank (apply max rank-lst)])
      (map (lambda (x) (exact->inexact (/ x max-rank))) rank-lst)))

> (-> gapminder
      (dataframe-filter
       (filter-expr (year) (= year 2007)))
      (->> ((lambda (df)
             (dataframe-modify
              df
              (modify-expr (percentile () (rank% ($ df 'gdpPercap))))))))
      (dataframe-sort (sort-expr (> gdpPercap)))
      (dataframe-display))

>  dim: 142 rows x 7 cols
             country  continent   year  lifeExp       pop  gdpPercap  percentile 
              Norway     Europe  2007.  80.1960  4.628E+6   49357.19      1.0000 
              Kuwait       Asia  2007.  77.5880  2.506E+6   47306.99      0.9930 
           Singapore       Asia  2007.  79.9720  4.553E+6   47143.18      0.9859 
       United States   Americas  2007.  78.2420  3.011E+8   42951.65      0.9789 
             Ireland     Europe  2007.  78.8850  4.109E+6   40676.00      0.9718 
  "Hong Kong, China"       Asia  2007.  82.2080  6.980E+6   39724.98      0.9648 
         Switzerland     Europe  2007.  81.7010  7.555E+6   37506.42      0.9577 
         Netherlands     Europe  2007.  79.7620  1.657E+7   36797.93      0.9507 
              Canada   Americas  2007.  80.6530  3.339E+7   36319.24      0.9437 
             Iceland     Europe  2007.  81.7570  3.019E+5   36180.79      0.9366 
```
 
 If we create an intermediate `dataframe`, we get simpler syntax.

```
(define gapminder2007
  (dataframe-filter
   gapminder
   (filter-expr (year) (= year 2007))))

(-> (dataframe-modify
     gapminder2007
     (modify-expr (percentile () (rank% ($ gapminder2007 'gdpPercap)))))
    (dataframe-sort (sort-expr (> gdpPercap)))
    (dataframe-display))
```

## Conclusion

The main outcome of writing this post was that it led me to completely rewrite `dataframe-display` because it previously didn't handle large numbers well, which was made abundantly clear in working with the `gapminder` data. The new version of `dataframe-display` is much improved and I gained a greater appreciation for all of the decisions that go into how to print a representation of a data structure to the screen. One lingering example of the difficulties in writing a good implementation of `dataframe-display` can be seen in the output above with `"Hong Kong, China"`. The quotation marks are there to prevent the comma in the string from being interpreted as a field separator in the CSV file. If I would have written the file as tab-delimited, then `Hong Kong, China` would have displayed correctly. But, clearly, R handles that case without a problem even with CSV input whereas `dataframe-display` would need another layer of logic in the string processing to handle the extra quotation marks.

If you are not familiar with Scheme code, you might find it to be unacceptably verbose in the examples above and, especially, when compared to `dplyr` code. Because of Scheme's macro system, `dataframe` could be written in a more terse style. But I made the decision early on to stick to relatively simple macro usage and write `dataframe` in a way that I thought would be familiar to Scheme programmers. And that often involves more verbose code. For example, nearly all of the `dataframe` procedures have `dataframe` in the name, e.g., `dataframe-filter`, `dataframe-modify`, etc. This is following the example for hashtables, e.g., `hashtable-ref`, `hashtable-values`, etc. I also hope that experienced Scheme programmers can see that the `dataframe` macros mostly exist to reduce the number of times that `lambda` is written but the shape of the code should still feel familiar. While Scheme code might be more verbose in the small, I find it extremely expressive in the large because the core ideas compose so well.  

***

<a name="1"></a> [1] Chez Scheme is the only Scheme implementation that I use so I tend to use 'Scheme' and 'Chez Scheme' interchangeably. However, after a recent pull request from John Cowan to remove the Chez-isms, `dataframe` should now be more portable and work with other R6RS Scheme implementations.
