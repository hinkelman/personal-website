+++
title = "DSM2 interactive map"
date = 2019-03-24
[taxonomies]
categories = ["R", "Shiny"]
tags = ["DSM2", "leaflet"]
+++

The [Delta Simulation Model II (DSM2)](https://water.ca.gov/Library/Modeling-and-Analysis/Bay-Delta-Region-models-and-tools/Delta-Simulation-Model-II) is a hydrodynamic model used for [Sacramento-San Joaquin Delta](https://en.wikipedia.org/wiki/Sacramento%E2%80%93San_Joaquin_River_Delta) planning and management. When working with DSM2 output, I frequently need to look up the location of channels, nodes, and stations in the model. A map of all of the channels, nodes, and stations (and more) is provided with DSM2 as a [PDF file](/pdf/DSM2_Grid2.0.pdf), but that file is not easy to search.

<!-- more -->

Fortunately, folks in the Delta Modeling Section of the Department of Water Resources shared [shapefiles](https://github.com/fishsciences/dsm2-map/tree/master/shapefiles) of DSM2 channels, nodes, and stations with me. With the power of [Shiny](https://shiny.rstudio.com/) and the [leaflet package](https://rstudio.github.io/leaflet/), it is relatively simple to build a web-based, interactive map of DSM2 channels, nodes, and stations (embedded below) [[1]](#1).

The app [[2]](#2) allows for panning, zooming, and clicking on the map to identify channels, nodes, and stations. Channels are green lines, nodes are black circles, and stations are brown circles. If you already know the channel, node, or station, then you can select them via dropdown menus with search boxes, which highlights the selected feature in yellow. Because most of the Delta is tidal, the app also indicates which node represents the upstream end of the channel (i.e., a positive value of flow in DSM2 output indicates that flow is moving through a channel from the upstream to the downstream end).

<iframe width="725" height="800" scrolling="no" frameborder="no" src="https://fishsciences.shinyapps.io/dsm2-map/"> </iframe>

***

<a name="1"></a> [1] Shiny app is hosted on [https://fishsciences.shinyapps.io/dsm2-map/](https://fishsciences.shinyapps.io/dsm2-map/) and code is available through [GitHub](https://github.com/fishsciences/dsm2-map).

<a name="2"></a> [2] The code in the DSM2 map app is derived from this [example](https://www.r-bloggers.com/r-shiny-leaflet-using-observers/).