+++
title = "RPI and SOS in Scheme, Python, and Elixir"
date = 2024-09-02
[taxonomies]
tags = ["Scheme", "Python", "Elixir", "dataframe", "Polars", "Explorer"]
+++

Last spring, I played in a 3x3 basketball leage with 14 teams and only 6 regular-season games. The unbalanced schedule made me wonder if we would end up with wonky playoff seeding. I thought it would be fun to calculate the Rating Percentage Index (RPI) and Strength Of Schedule (SOS) for each team to assess discrepancies between W-L record and team rating. I was mostly following the R code for RPI in [this post](http://dpmartin42.github.io/posts/r/college-basketball-rankings) and the SOS calculations [here](https://hackastat.eu/en/learn-a-stat-strength-of-schedule-sos/). For comparison, I wrote code in Scheme ([dataframe](https://github.com/hinkelman/dataframe)), Python ([Polars](https://pola.rs/)), and Elixir ([Explorer](https://hexdocs.pm/explorer/Explorer.html)), which is also using [Polars](https://docs.rs/polars/latest/polars/) as the backend. All code and data are available [here](https://github.com/hinkelman/rpi). Disclaimer: the Python and Elixir code works, but it may not be the most idiomatic or performant way to write that code. 

<!-- more -->

First, let's load libraries and read the data.

```
;; Scheme dataframe
(import (dataframe))

(define df (csv->dataframe "GameResults.csv"))

# Python Polars
import polars as pl
import statistics as stats

df = pl.read_csv("GameResults.csv")

# Elixir Explorer
require Explorer.DataFrame, as: DF
require Explorer.Series, as: Series

df = DF.from_csv!("GameResults.csv")
```

Every game was comprised of three sets to 21 points (win by two points). The league tracked W-L record by game, but I used sets to increase the amount of data. The actual RPI calculation includes a weighting factor for home and away games, but all of these games were played at a neutral site, which simplified the calculations. There were several forfeits, which the league tracked as losses, but I decided to exclude them from my dataset. 

This is the format of the raw data...

```
> (dataframe-display df)
 dim: 132 rows x 6 cols
    week     set       winner         loser  winner_score  loser_score 
   <num>   <num>        <str>         <str>         <num>        <num> 
      1.      1.   Above ParR  CityOfThrees           22.          16. 
      1.      2.   Above ParR  CityOfThrees           23.          21. 
      1.      3.   Above ParR  CityOfThrees           21.          15. 
      1.      1.        Kangz          Tsaf           21.          17. 
      1.      2.         Tsaf         Kangz           21.          19. 
      1.      3.        Kangz          Tsaf           21.          11. 
      1.      1.  Team Avatar        Motley           22.          14. 
      1.      2.       Motley   Team Avatar           21.           0. 
      1.      3.       Motley   Team Avatar           23.          21. 
      1.      1.     Chow Men      Bob Ross           21.          10. 
```

The approach to calculating W-L record involves two passes through the data, which could prove costly with a larger dataset. The `type` parameter refers to the `winner` or `loser` column in the dataframe and is passed as a symbol (Scheme), string (Python), or atom (Elixir). Polars and Explorer both use square brackets for extracting a dataframe column whereas `$` is used in Scheme dataframe.

```
;; Scheme dataframe
(define (calc-wl game-data team type)
  (length (filter (lambda (x) (string=? team x)) ($ game-data type))))

# Python Polars
def calc_wl(game_data, team, type):
  return pl.Series.sum(game_data[type] == team)

# Elixir Explorer
def calc_wl(game_data, team, type) do
  Series.sum(Series.equal(game_data[type], team))
end
```

All of the calculations involve filtering the dataset to the focal `team` and potentially excluding games played against another team (`opp`). Perhaps the most notable difference here (other than the verbosity of the Scheme code) is the different ways that dataframe columns are referenced. Scheme dataframe and Explorer use unquoted column names. Polars uses the `col` function and quoted names. In Explorer, a query expression expects unquoted names to refer to dataframe columns and the `^` is needed to indicate that a variable is defined outside of the query (e.g., `^team` and `^opp`). In Scheme dataframe, column names are specified separately (i.e., `(winner loser)`) to create the distinction between column names and other variables. This approach was taken to mirror lambda syntax, but maybe I should explore whether I could use the `^` inside the filter expression to avoid the redundant listing of column names.

```
;; Scheme dataframe
(define (filter-team game-data team)
  (dataframe-filter*
   game-data
   (winner loser)
   (or (string=? winner team)
       (string=? loser team))))

(define (filter-team-opp game-data team opp)
  (dataframe-filter*
   (filter-team game-data team)
   (winner loser)
   (and (not (string=? winner opp))
        (not (string=? loser opp)))))

# Python Polars
def filter_team(game_data, team):
    return pl.DataFrame.filter(
        game_data, 
        (pl.col("winner") == team) | (pl.col("loser") == team))


def filter_team_opp(game_data, team, opp):
    return pl.DataFrame.filter(
        filter_team(game_data, team),
        (pl.col("winner") != opp) & (pl.col("loser") != opp))

# Elixir Explorer
def filter_team(game_data, team) do
  DF.filter(game_data, winner == ^team or loser == ^team)
end

def filter_team_opp(game_data, team, opp) do
  DF.filter(filter_team(game_data, team), winner != ^opp and loser != ^opp)
end
```

I've arguably opted to split the code into too many function as indicated by the difficulty I had in naming this next function. This code takes the dataframe column of winners and finds the proportion that match the specified team. This function is only applied to a filtered dataset for the specified team.

```
;; Scheme
(define (wp winners team)
  (let ([team-winners (filter (lambda (x) (string=? team x)) winners)])
    (inexact (/ (length team-winners) (length winners)))))

# Python Polars
def wp(winners, team):
  return pl.Series.sum(winners == team) / pl.Series.count(winners)

# Elixir Explorer
def wp(winners, team) do
  Series.sum(Series.equal(winners, team)) / Series.count(winners)
end
```

In `calc-wp`, we are combining our previous functions so the code is similar in all languages. 

```
;; Scheme dataframe
(define (calc-wp game-data team)
  (let ([games-played (filter-team game-data team)])
    (wp ($ games-played 'winner) team)))

# Python Polars
def calc_wp(game_data, team):
  games_played = filter_team(game_data, team)
  return wp(games_played["winner"], team)

# Elixir Explorer
def calc_wp(game_data, team) do
  games_played = filter_team(game_data, team)
  wp(games_played[:winner], team)
end
```

I was also interested in point differential (PD) as a metric, but it became clear as the season wore on that the scores were not very accurate. The league used set differential as a tiebreaker so the scores were largely irrelevant as long as they recorded the correct set winners and losers. I've included two versions of the Scheme code for comparison. While legible, I find the Polars code a bit ugly here. 

```
;; Scheme
(define (calc-pd game-data team)
  (let* ([games-played (filter-team game-data team)])
    (sum (map (lambda (w ws ls) (if (string=? team w) (- ws ls) (- ls ws)))
              ($ games-played 'winner)
              ($ games-played 'winner_score)
              ($ games-played 'loser_score)))))

;; Scheme dataframe
(define (calc-pd game-data team)
  (-> game-data
      (filter-team team)
      (dataframe-modify*
       (pd (winner winner_score loser_score)
           (if (string=? team winner)
               (- winner_score loser_score)
               (- loser_score winner_score))))
      ($ 'pd)
      (sum)))

# Python Polars
def calc_pd(game_data, team):
    pd = pl.DataFrame.with_columns(
        filter_team(game_data, team),
        pl.when(pl.col("winner") == team)
        .then(pl.col("winner_score") - pl.col("loser_score"))
        .otherwise(pl.col("loser_score") - pl.col("winner_score"))
        .alias("pd"))
    return pl.Series.sum(pd["pd"])

# Elixir Explorer
  def calc_pd(game_data, team) do
    game_data
    |> filter_team(team)
    |> DF.mutate(
      pd: if(winner == ^team, 
        do: winner_score - loser_score, 
        else: loser_score - winner_score)
    )
    |> DF.pull("pd")
    |> Series.sum()
  end
```

In calculating the opponents' winning percentage, we need to exclude games played against the focal team with `filter-team-opp`. The way the parameters are named is potentially confusing. In `(map (lambda (x) (calc-wp-owp game-data x team)) opps)`, we are iterating through all opponents of `team` and calculating their WP against everybody except `team` and then calculating the mean of all opponents' WP. The function for `calc-oowp` is is nearly the same as `calc-owp` so is not shown here.

```
;; Scheme
(define (calc-owp game-data team)
  (let* ([opp-games (filter-team game-data team)]
         [opps (map (lambda (w l) (if (string=? team w) l w))
                    ($ opp-games 'winner)
                    ($ opp-games 'loser))]
         [owp (map (lambda (x) (calc-wp-owp game-data x team)) opps)])
    (mean owp)))

(define (calc-wp-owp game-data team opp)
  (let ([games-played (filter-team-opp game-data team opp)])
    (wp ($ games-played 'winner) team)))

# Python Polars
def calc_owp(game_data, team):
    opp_games = pl.DataFrame.filter(
        game_data, (pl.col("winner") == team) | (pl.col("loser") == team))
    opp = pl.DataFrame.with_columns(
        opp_games,
        pl.when(pl.col("winner") == team)
        .then(pl.col("loser"))
        .otherwise(pl.col("winner"))
        .alias("opp"))
    return stats.mean([calc_wp_owp(game_data, x, team) for x in opp["opp"]])

def calc_wp_owp(game_data, team, opp):
  games_played = filter_team_opp(game_data, team, opp)
  return wp(games_played["winner"], team)

# Elixir Explorer
def calc_owp(game_data, team) do
  game_data
  |> DF.filter(winner == ^team or loser == ^team)
  |> DF.mutate(opp: if(winner == ^team, do: loser, else: winner))
  |> DF.pull("opp")
  # transform is computationally expensive b/c of type conversion
  |> Series.transform(fn x -> calc_wp_owp(game_data, x, team) end)
  |> Series.mean()
end

def calc_wp_owp(game_data, team, opp) do
  games_played = filter_team_opp(game_data, team, opp)
  wp(games_played[:winner], team)
end
```

The next functions are simple arithmetic, but are included to show Scheme's prefix mathematical operations and how the `\` is used in Python to break an operation across lines.

```
;; Scheme
(define (calc-sos game-data team)
  (/ (+ (* 2 (calc-owp game-data team)) (calc-oowp game-data team)) 3))

(define (calc-rpi game-data team)
  (+ (* 0.25 (calc-wp game-data team))
     (* 0.5 (calc-owp game-data team))
     (* 0.25 (calc-oowp game-data team))))

# Python
def calc_sos(game_data, team):
  return (2 * calc_owp(game_data, team) + calc_oowp(game_data, team)) / 3

def calc_rpi(game_data, team):
  rpi = 0.25 * calc_wp(game_data, team) + \
    0.5 * calc_owp(game_data, team) + \
    0.25 * calc_oowp(game_data, team)
  return rpi

# Elixir
def calc_sos(game_data, team) do
  (2 * calc_owp(game_data, team) + calc_oowp(game_data, team)) / 3
end

def calc_rpi(game_data, team) do
  0.25 * calc_wp(game_data, team) +
    0.5 * calc_owp(game_data, team) +
    0.25 * calc_oowp(game_data, team)
end
```

We need a list of all teams for iteration. Even though both Polars and Explorer are using Rust polars as a backend, my understanding is that Python Polars is using similar naming to Rust polars, but Explorer is creating an API that is more similar to [R dyplr](https://dplyr.tidyverse.org/) (e.g., unique vs distinct). Scheme dataframe returns a list whereas Polars and Explorer both return Series. The Scheme list is used in a mapping operation and the Polars Series can be used in a list comprehension, but the Explorer Series is converted to an enumerable for mapping.

```
;; Scheme
(define teams (remove-duplicates (append ($ df 'winner) ($ df 'loser))))

# Python Polars
teams = df["winner"].append(df["loser"]).unique()

# Elixir Explorer
teams =
  Series.concat(df[:winner], df[:loser])
  |> Series.distinct()
  |> Series.to_enum()
```

Finally, we put it all together and create a dataframe with all of our calculated columns sorted by descending RPI. The process is similar in all languages (but, again, most verbose in Scheme) and involves mapping (Scheme, Elixir) or list comprehensions (Python) to iterate over all the teams. In Scheme and Python, we set the number of rows displayed equal to the number of teams. I ran the Elixir code in Livebook and interactively changed the number of rows to display. 

I made no effort to write performant code because the dataset is so small, but the Python and Elixir code take 2-3 seconds on my machine while the Scheme code is effectively instanteous. I'm assuming that is mostly overhead of dealing with the Rust polars backend and would be a neglible element of the overall compute time with a larger dataset. 

```
;; Scheme dataframe
(-> (make-dataframe
     (list (make-series 'Team teams)
           (make-series 'Win (map (lambda (x) (calc-wl df x 'winner)) teams))
           (make-series 'Loss (map (lambda (x) (calc-wl df x 'loser)) teams))
           (make-series 'WP (map (lambda (x) (calc-wp df x)) teams))
           (make-series 'PD (map (lambda (x) (calc-pd df x)) teams))
           (make-series 'SOS (map (lambda (x) (calc-sos df x)) teams))
           (make-series 'RPI (map (lambda (x) (calc-rpi df x)) teams))))
    (dataframe-sort* (> RPI))
    (dataframe-display (length teams)))
    
# Python Polars
pl.Config.set_tbl_rows(len(teams))
pl.DataFrame(
  {
    "team": teams,
    "win": [calc_wl(df, x, "winner") for x in teams],
    "loss": [calc_wl(df, x, "loser") for x in teams],
    "wp": [calc_wp(df, x) for x in teams],
    "pd": [calc_pd(df, x) for x in teams],
    "sos": [calc_sos(df, x) for x in teams],
    "rpi": [calc_rpi(df, x) for x in teams]
  }
).sort("rpi", descending = True)

# Elixir Explorer
DF.new(
  team: teams,
  win: Enum.map(teams, fn x -> RPI.calc_wl(df, x, :winner) end),
  loss: Enum.map(teams, fn x -> RPI.calc_wl(df, x, :loser) end),
  wp: Enum.map(teams, fn x -> RPI.calc_wp(df, x) end),
  pd: Enum.map(teams, fn x -> RPI.calc_pd(df, x) end),
  sos: Enum.map(teams, fn x -> RPI.calc_sos(df, x) end),
  rpi: Enum.map(teams, fn x -> RPI.calc_rpi(df, x) end)
)
|> DF.sort_by(desc: rpi)
```

Here were the RPI rankings at the end of the regular season:

```
 dim: 14 rows x 7 cols
             Team     Win    Loss      WP      PD     SOS     RPI 
            <str>   <num>   <num>   <num>   <num>   <num>   <num> 
         Spartans     16.      2.  0.8889    141.  0.5971  0.6701 
         Chow Men     14.      4.  0.7778    103.  0.5274  0.5900 
  The Free Agents     11.      7.  0.6111     59.  0.5385  0.5567 
             >30%      8.      7.  0.5333     -7.  0.5464  0.5432 
       Above ParR      8.      7.  0.5333    -15.  0.5364  0.5357 
        The Big 3     11.      7.  0.6111     20.  0.4905  0.5206 
     Net Positive      6.      6.  0.5000    -10.  0.4956  0.4967 
         Bob Ross      8.     10.  0.4444    -23.  0.4968  0.4837 
           Motley      8.      7.  0.5333     17.  0.4603  0.4786 
       The A-Team      1.      8.  0.1111    -83.  0.5855  0.4669 
      Team Avatar      9.      9.  0.5000    -24.  0.4242  0.4431 
            Kangz      7.     11.  0.3889    -12.  0.4501  0.4348 
             Tsaf      4.     14.  0.2222    -92.  0.4563  0.3978 
     CityOfThrees      3.     15.  0.1667    -74.  0.4432  0.3741 
```

The top 3 ranked teams were also seeded as teams 1-3 in the playoffs based only on game record and set differential (not set record, RPI, etc.). >30% was seeded at #11 because they forfeited twice, which I didn't attempt to account for. Team Avatar was the #6 seed because they benefited from forfeits that aren't reflected in the rankings. Apart from the effects of forfeits, the RPI rankings for each team were generally within 1 or 2 places from the playoff seeding. I was surprised that only six regular season games produced playoff seedings that so reasonably reflected the teams' rankings. 