+++
title = "Updating Shiny Scorekeeper app with Shiny modules and bslib"
date = 2025-04-27
[taxonomies]
tags = ["R", "Shiny"]
+++

[Shiny Scorekeeper](https://github.com/hinkelman/Shiny-Scorekeeper) is a basketball scorekeeper app built with the [Shiny](https://shiny.posit.co) web framework for [R](https://www.r-project.org). The app was initially built in 2018, but I recently decided to update it to improve maintainability and provide a more modern look. It's sort of a strange choice to invest time in this project in 2025. I built the app for scoring videos of my son's youth basketball games, but my son is no longer a youth and I haven't used the app in years. However, I have a fondness for the app because I had fun building it initially and it served me well for the years that I used it.

<!-- more -->

This new version is **not** about adding new features. The core functionality in the app is relatively unchanged, e.g., I'm still using [editable DataTables from the `DT` package](/dt-datatable-crud/) to create and manage rosters. One of the big changes was splitting the code among [Shiny modules](https://mastering-shiny.org/scaling-modules.html) and an [R package](https://github.com/hinkelman/scorekeepeR). In the previous version, the `server.R` file was over 800 lines of code that I didn't find very easy to maintain or extend.

I moved some of the code into an R package partly because I've recently had fun playing with Tk in R and Scheme (see [this post](/eda-scheme-tk/)) and thought it might be fun to try to build a version of the app with a Tk interface (as an alternative to the Shiny interface). I had plenty of moments, though, where I regretted adding the extra overhead of having that code in a separate R package, especially considering the payoff is mostly hypothetical and delayed (e.g., future maintenance or different interface). I also didn't make any effort to think about whether the functions in the package are written in a general way that will work equally well with a Tk interface.

Despite building many Shiny apps over the years, including some reasonably complex apps, I had never used Shiny modules. One one of the primary objectives in this Shiny Scorekeeper refresh project was to learn how to use Shiny modules. My use of modules here was limited to the simplest use case, though, i.e., splitting code among files. The app is comprised of three sections (Roster, Scorekeeper, Stats Viewer) that led to obvious breaks for splitting into modules. 

The biggest friction that I encountered with using modules is that auto reload is not triggered by changes to module files. Fortunately, the auto reload behavior will be included in a future version of Shiny (see [this pull request](https://github.com/rstudio/shiny/pull/4184)). The most common mistake that I made with modules was forgetting to [namespace UI elements](https://mastering-shiny.org/scaling-modules.html#namespacing), which often produces unexpected behavior without any errors sent to the R console. The only other aspect that I found a little tricky was understanding how to pass values around among the modules. 

The other big change was switching from [`shinydashboard`](https://rstudio.github.io/shinydashboard/) to [`bslib`](https://rstudio.github.io/bslib/). Design is not one of my strengths so I tend to stick with default themes and colors. By default, `shinydashboard` uses too much color for my liking. 

![image of old version of scorekeeper](/img/scorekeeper-old.png)

The `bslib` default uses very little color, which looks modern to me, but `bslib` also provides many [themes](https://rstudio.github.io/bslib/articles/theming/index.html). 

![image of new version of scorekeeper](/img/scorekeeper-new.png)

I've started using `bslib` in nearly all of my Shiny apps. The default layouts look better than stock Shiny (or `shinydashboard`) apps and the elements compose nicely to create more complex layouts. I think the new `bslib` version of Shiny Scorekeeper uses space more effectively, which followed naturally from the different default layout options (e.g., panel navigation in the top bar instead of the sidebar for `page_navbar`).

I decided to stick to more of the standard Shiny UI components in this version. That was partly because I perceived a more uniform look with those elements when using `bslib`. The one 3rd-party component that I couldn't give up was `shinyWidgets::pickerInput`. When using multiple select, the Shiny alternative (`selectInput`) is uglier and provides far fewer options (e.g., select/deselect all, live search, etc.). 

The Stats Viewer in Shiny Scorekeeper changed more than any other section. Previously, I used `DT` tables as the mechanism for sorting and filtering the team and game data (by selecting rows in the tables). This approach allowed for compact presentation of the details associated with each team and game, but provided an unconventional interface that I ended up not liking. 

![image of old version of stats viewer](/img/stats-old.png)

For the new version, I have hierarchical inputs to filter down to the desired game stats. This approach leads to a relatively large hierarchy: leagues -> teams -> seasons -> scoring margin -> opponents -> dates. Changes at the top of the hierarchy are propagated dynamically to update all of the other inputs. This requires more code than my previous approach with tables and could lead to notable lag in updating of the game stats table as data size increases.

![image of new version of stats viewer](/img/stats-new.png)

I also replaced the `DT` game stats table with a [`reactable`](https://glin.github.io/reactable/) version. `DT` tables have been my go-to option for Shiny apps for many years, but I recently read that the [`DT` author thinks `reactable` is generally better than `DT`](https://bookdown.org/yihui/rmarkdown-cookbook/table-other.html). I would have used `reactable` for all tables in Shiny Scorekeeper, but `reactable` tables are not editable so I needed to stick with `DT` tables for the roster section. I also think it is nice that the game stats table has a different look to reinforce that it is not editable. My only gripe with the `reactable` table is that I didn't like the default behavior for setting column widths so I had to write extra code to handle that. 




