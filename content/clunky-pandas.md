+++
title = "Comparing dataframe operations in Scheme, Python, and R"
date = 2024-03-27
[taxonomies]
categories = ["Scheme", "Chez Scheme", "dataframe"]
tags = ["dataframe", "dplyr", "pandas"]
+++

I recently came across [this blog post](https://www.sumsar.net/blog/pandas-feels-clunky-when-coming-from-r/) that calls [`pandas`](https://pandas.pydata.org/) (Python) "clunky" compared to the "silky smooth" [`dplyr`](https://dplyr.tidyverse.org/) (R). No objections from me. `dplyr` is my favorite R package. I thought it would fun to compare the relative clunkiness of my [`dataframe`](https://github.com/hinkelman/dataframe/) library for Scheme (R6RS) to `pandas` and `dplyr`. I will include the R and Python code here to save some clicking back and forth, but will mostly be commenting on the Scheme code.

<!-- more -->

## Reading in the data

```
# R 
library(dplyr)
purchases <- read.csv("purchases.csv")

# Python
import pandas as pd
purchases = pd.read_csv("purchases.csv")

;; Scheme
(import (dataframe))
(define purchases (csv->dataframe "purchases.csv"))
```

## "How much do we sell..? Let's take the total sum!"

```
# R 
sum(purchases$amount)

# Python
sum(purchases["amount"])

;; Scheme
(sum ($ purchases 'amount))
```

## "Ah, they wanted it by country..."

`dataframe-aggregate*` is a macro to slightly reduce the verbosity of the Scheme code, but it still falls short of the brevity and readability of the `dplyr` code. 

```
# R 
purchases |>
  group_by(country) |>
  summarize(total = sum(amount))

# Python
(purchases
  .groupby("country")
  .agg(total=("amount", "sum")) 
  .reset_index()                
)

;; Scheme
(dataframe-aggregate* purchases (country) (total (amount) (sum amount)))
```

## "And I guess I should deduct the discount."

In Version 1, subtracting the discount is included by mapping over the columns within `dataframe-aggregate*`. In Version 2, `dataframe-modify*` handles mapping down the columns and produces more compact code. Version 2 illustrates the similarities in `dataframe-modify*` and `dataframe-aggregate*` with the primary difference being that you need to specify the grouping columns, e.g., `(country)` in `dataframe-aggregate*`, but the general form is `(new-name (names) (expression))`. 

```
# R 
purchases |> 
  group_by(country) |> 
  summarize(total = sum(amount - discount))

# Python
(purchases
  .groupby("country")
  .apply(lambda df: (df["amount"] - df["discount"]).sum()) 
  .reset_index()
  .rename(columns={0: "total"})                            
)

;; Scheme
;; Version 1
(-> purchases
    (dataframe-aggregate*
     (country)
     (total
      (amount discount)
      (sum (map (lambda (amt dis) (- amt dis)) amount discount)))))

;; Version 2
(-> purchases
    (dataframe-modify* (diff (amount discount) (- amount discount)))
    (dataframe-aggregate* (country) (total (diff) (sum diff))))
```

## “Oh, and Maria asked me to remove any outliers.”

The Scheme version only works because the filter step is the first in the pipeline, i.e., the pipe is passing the same `purchases` that is referred to in `($ purchases 'amount)`. 

```
# R 
purchases |>
  filter(amount <= median(amount) * 10) |> 
  group_by(country) |> 
  summarize(total = sum(amount - discount))

# Python
(purchases
  .query("amount <= amount.median() * 10")
  .groupby("country")
  .apply(lambda df: (df["amount"] - df["discount"]).sum())
  .reset_index()
  .rename(columns={0: "total"})
)

;; Scheme
(-> purchases
    (dataframe-filter* (amount) (<= amount (* 10 (median ($ purchases 'amount)))))
    (dataframe-modify* (diff (amount discount) (- amount discount)))
    (dataframe-aggregate* (country) (total (diff) (sum diff))))
```

## “I probably should use the median within each country.”

In Version 1, the outlier filtering and discount subtraction happens within the hidden split-apply-combine approach in `dataframe-aggregate*`. In Version 2, the split-apply-combine is made explicit for outlier filtering. Personally, I find Version 2 more readable, but it should be avoided for large dataframes because it involves splitting twice (`dataframe-split` and `dataframe-aggregate*`). Of course, neither are as elegant as the `dplyr` version.

```
# R 
purchases |>
  group_by(country) |>                    
  filter(amount <= median(amount) * 10) |> 
  summarize(total = sum(amount - discount))

# Python
(purchases
  .groupby("country")                                               
  .apply(lambda df: df[df["amount"] <= df["amount"].median() * 10]) 
  .reset_index(drop=True)                                           
  .groupby("country")
  .apply(lambda df: (df["amount"] - df["discount"]).sum())
  .reset_index()
  .rename(columns={0: "total"})
)

;; Scheme
;; Version 1
(-> purchases
    (dataframe-aggregate*
     (country)
     (total
      (amount discount)
      (sum (map (lambda (amt dis)
                  (let ([multi (if (<= amt (* 10 (median amount))) 1 0)])
                    (* multi (- amt dis))))
                amount
                discount)))))

;; Version 2
(-> purchases
    (dataframe-split 'country)
    (->> (map (lambda (dfx)
                (dataframe-filter* dfx (amount) (<= amount (* 10 (median ($ dfx 'amount))))))))
    (dataframe-bind-all)
    (dataframe-modify* (diff (amount discount) (- amount discount)))
    (dataframe-aggregate* (country) (total (diff) (sum diff))))
```

## Conclusion

I'm clearly biased, but I think a case could be made that my `dataframe` library for Scheme is at least not more clunky than `pandas`.
