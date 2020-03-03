---
date: "2018-11-21"
title: DSM2 HYDRO Visualization Tool
type: section
---

<br>
<img src="/img/dsm2-viz-tool.png" width="100%"/>

The [Sacramento-San Joaquin Delta](https://sacdeltaguide.atavist.com/) is an inland freshwater estuary subject to intensive management. The [Delta Simulation Model II (DSM2)](http://baydeltaoffice.water.ca.gov/modeling/deltamodeling/models/dsm2/dsm2.cfm) is one of the most commonly used hydrodynamic models for Delta planning and management with modules for hydrodynamics (HYDRO), water quality (QUAL), and particle tracking (PTM). The California Department of Water Resources uses DSM2 for planning and operation of the State Water Project (SWP) and Central Valley Project (CVP) diversion pumping as well as for management and satisfaction of stringent water quality standards. Despite the widespread use of DSM2, we are not aware of any tools for interactively visualizing DSM2 HYDRO output. An easy-to-use visualization tool expands the accessibility of DSM2 output beyond hydrodynamic modelers to include biologists, water managers, and water users.

Our initial work focused on visualizing the effects of inflow and exports on large-scale hydrodynamic patterns across numerous Delta channels. One of our motivating questions was how to identify the ‘footprint’ of diversion pumping at the SWP and CVP facilities. Those interests led us to an interactive, map-based approach where the map both displays large-scale patterns and allows for navigation among channels to display channel-level details. We used the [Shiny](https://shiny.rstudio.com/) web framework to build a [Delta Hydrodynamics app](https://fishsciences.shinyapps.io/delta-hydrodynamics/) that provides interactivity with our analysis of outputs from a set of DSM2 runs. The [DSM2 HYDRO Visualization Tool (DSM2 Viz Tool)](https://github.com/fishsciences/DSM2-Viz-Tool) is a general tool that allows for uploading DSM2 output files to create similar visualizations as used in the Delta Hydrodynamics app.

If you want to learn more about the DSM2 Viz Tool, you can [install it](https://github.com/fishsciences/DSM2-Viz-Tool) and play around with the example files. If you just want a quick peek at the core ideas, then take a look at the [Delta Hydrodynamics app](https://fishsciences.shinyapps.io/delta-hydrodynamics/). We are currently working on a short manuscript that describes the DSM2 Viz Tool, which will become the natural starting point for learning about the tool.

We did not set out to build a general DSM2 visualization tool. Rather, we were following our usual workflow of using R to analyze data and Shiny to make our analysis interactive and accessible. The use of R and Shiny to make a general DSM2 visualization tool is arguably a case of [Maslow's hammer](https://en.wikipedia.org/wiki/Law_of_the_instrument) because a web application is an unusual choice for a tool that involves interaction with the typically large files produced by DSM2. However, [Electron](https://electronjs.org/) allows us to turn our Shiny app into a standalone desktop application (described [here](/post/deploy-shiny-electron/)). 

### File formats

DSM2 produces output in two file formats: [HEC-DSS](http://www.hec.usace.army.mil/software/hec-dss/) and [HDF5](https://portal.hdfgroup.org/display/HDF5/HDF5). The HEC-DSS format is the older and more commonly used format, but the R package ([DSS-Rip](https://github.com/eheisman/dssrip)) for working with HEC-DSS files only works with 32-bit R on a Windows machine. The R packages ([hdf5r](https://github.com/hhoeflin/hdf5r) and [rhdf5](https://bioconductor.org/packages/release/bioc/html/rhdf5.html)) for working with HDF5 files don't have the same constraints as DSS-Rip. We chose to use rhdf5 for the DSM2 Viz Tool because the HDF5 libaries are provided as an R package ([Rhdf5lib](https://bioconductor.org/packages/release/bioc/html/Rhdf5lib.html)) rather than requiring installation.  

### Dependencies
* [shiny](http://shiny.rstudio.com)
* [shinyWidgets](https://dreamrs.github.io/shinyWidgets/index.html)
* [shinyFiles](https://github.com/thomasp85/shinyFiles)
* [shinyjs](https://deanattali.com/shinyjs/)
* [leaflet](https://rstudio.github.io/leaflet/)
* [DT](https://rstudio.github.io/DT/)
* [rhdf5](https://www.bioconductor.org/packages/release/bioc/html/rhdf5.html)
* [Rhdf5lib](https://bioconductor.org/packages/release/bioc/html/Rhdf5lib.html)
* [rgdal](https://cran.r-project.org/web/packages/rgdal/index.html)
* [dplyr](https://dplyr.tidyverse.org)
* [plyr](http://plyr.had.co.nz)
* [tidyr](https://tidyr.tidyverse.org)
* [lubridate](https://lubridate.tidyverse.org)
* [ggplot2](https://ggplot2.tidyverse.org)
* [scales](https://scales.r-lib.org)
* [Cairo](https://cran.r-project.org/web/packages/Cairo/index.html)
* [fs](https://fs.r-lib.org)
* [MESS](https://github.com/ekstroem/MESS)
