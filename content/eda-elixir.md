+++
title = "Exploratory data analysis in Elixir"
date = 2023-07-08
[taxonomies]
categories = ["Elixir", "dataframe", "Kino", "Explorer"]
tags = ["EDA", "dataframe"]
+++

I have been tinkering with lots of different programming languages (see [here](/programming-horizons/) and [here](/programming-horizons-revisited/)) over the last few years. Scheme is the only language so far that I have enjoyed enough to write a decent amount of code. [Elixir](https://elixir-lang.org/) first caught my eye back in April 2020, but I've only recently tried to write more than 'hello world' with it. So far, I think it is great and I'm excited to learn more. I haven't previously been a fan of code notebooks, but I think [Livebook](https://livebook.dev/) is amazing.

<!-- more -->

I'm using Elixir on Ubuntu. There are lots of different ways to install Elixir on Linux. I opted to first install Erlang with `sudo apt install erlang`. Then I installed the [Elixir precompiled package](https://elixir-lang.org/install.html#precompiled-package) for the version of Erlang that I just installed (get Erlang/OTP version by running `erl -s halt`). I moved the unzipped folder to my home directory and added the following line to `.zshrc`: `export PATH="$PATH:/home/username/elixir-otp-25/bin"`.

I installed Livebook by running `mix escript.install hex livebook` in the terminal and added the following line to `.zshrc`: `export PATH="$PATH:/home/username/.mix/escripts"`. After running `source .zshrc`, I was able to launch Livebook with `livebook server`. 

As with learning Scheme, I like to first try to recreate examples that I've written in other programming languages. Below, I've written Elixir code that corresponds to the Scheme examples in [this blog post](https://www.travishinkelman.com/eda-scheme/) based on the Texas housing dataset that is included as part of the `ggplot2` package for R. I wrote that post to try out my `dataframe` library for Scheme, but below I will focus on the comparison with R. I will highlight snippets of code in this post, but the full notebook is [here](https://github.com/hinkelman/livebook/blob/main/txhousing.livemd).

One of the great features of Livebook is that the user-friendly GUI elements create code that is then easy to edit. It dramatically lowers the learning curve for beginners. At the top of every notebook is a setup block that includes an `Add package` button that allows for searching of available packages and generates the following code.

```
Mix.install([
  {:kino, "~> 0.9.4"},
  {:kino_explorer, "~> 0.1.8"},
  {:kino_vega_lite, "~> 0.1.8"},
  {:req, "~> 0.3.10"}
])
```

`Kino` is behind the magic of Livebook. `KinoExplorer` and `KinoVegaLite` provide nice tables for viewing dataframes and data visualizations, respectively. Those packages include `Explorer` and `VegaLite` as dependencies so they do not need to be installed separately. I'm using `req` to get data directly from a URL. 

[`Explorer`](https://hexdocs.pm/explorer/Explorer.html) is a dataframe library for Elixir. I love the choice to mostly follow `dplyr`, which is my favorite R package. We `require` these packages because they require compilation and include aliases with `as:`.

```
require Explorer.DataFrame, as: DF
require Explorer.Series, as: Series
```

In R, `read.csv` allows for reading directly from a URL. This Elixir code composes well, though. It didn't occur to me at first (but should have) that I should be searching for how to do a `GET` request. `load_csv` is used here because there is already a representation of the CSV in memory. To read a file from disk, use `from_csv`. The `!` in `load_csv!` indicates that a problem with the file will raise an exception, which is arguably the preferred behavior for an interactive use case like this notebook. 

```
txhousing =
  Req.get!("https://www.travishinkelman.com/data/txhousing.csv").body
  |> DF.load_csv!()
```

This block of code is almost exactly the same as what you would write in `dplyr`. I love that Elixir has a pipe operator (and it even uses the same characters as the pipe operator in base R!). It looks a little weird that pipe operators are placed at the beginning of lines after many years of using pipes in R.

```
df_agg_year =
  txhousing
  |> DF.group_by("year")
  |> DF.summarise(
    avg_sales: mean(sales),
    avg_volume: mean(volume),
    avg_median: mean(median)
  )
```

In `group_by`, the column `month` can be written as a string (`"month"`) or an atom (`:month`) but it can't be bare (`month`). In `summarise` and `arrange`, the column names must be bare, not a string or atom. Without reading the documentation or source code, I would guess that `summarise` and `arrange` are macros and `group_by` is not.

```
df_agg_month =
  txhousing
  |> DF.group_by(:month)
  |> DF.summarise(
    avg_sales: mean(sales),
    avg_volume: mean(volume),
    avg_median: mean(median)
  )
  |> DF.arrange(month)
```

One of the convenient features of `dplyr::mutate` is that expressions can perform calculations on columns that were created within that `mutate`. Here we need to move that last expression to its own `DF.mutate`.

```
txhousing
|> DF.group_by(["city", "year"])
|> DF.mutate(
  total_sales: sum(sales),
  total_volume: sum(volume)
)
# need a new mutate when working with newly calculated column
|> DF.mutate(prop_sales: sales / total_sales)
```