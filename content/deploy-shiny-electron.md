+++
title = "Deploying a Shiny app as a desktop application with Electron"
date = 2018-11-21
updated = 2020-11-28
[taxonomies]
tags = ["R", "Shiny", "Electron"]
+++

In developing the [DSM2 HYDRO Viz Tool](https://github.com/fishsciences/dsm2-viz-tool), we were faced with deciding how to deploy a Shiny app that required interaction with large local files. I first heard about the possibility of using [Electron](https://electronjs.org/) to deploy Shiny apps as standalone desktop applications in this [talk by Katie Sasso](https://www.youtube.com/watch?v=ARrbbviGvjc), but it wasn't until I discovered the [R Shiny Electron (RSE) template](https://github.com/dirkschumacher/r-shiny-electron) that I decided to take the plunge. 

<!-- more -->

UPDATE (2020-09-06): The approach described in this post is fragile and I no longer use it. Here are a couple of alternatives: [electricShine](https://chasemc.github.io/electricShine/) and [photon](https://github.com/ColumbusCollaboratory/photon).

UPDATE (2020-11-28): I spent some time this weekend looking again at options for using Electron to deploy Shiny apps. My current conclusion is that it is more trouble than it is worth. Instead, I recommend [DesktopDeployR](https://github.com/wleepang/DesktopDeployR) as a simpler solution (Windows only). The end result is less slick than an Electron app, but involves much less hassle.

## R Shiny Electron Build Process

The [R Shiny Electron (RSE) template](https://github.com/dirkschumacher/r-shiny-electron) includes only very basic instructions for how to get started with this process (and they clearly specify that it is not ready for production). There were considerable gaps in my understanding that left me flailing while trying to get things to work (especially on Windows). Below I am describing what I did to get up and running but I make no argument that this process represents best practices.

### Basic setup

The RSE template doesn't mention anything about the arguably obvious steps of installing [Node](https://nodejs.org/), [Electron](https://electronjs.org/), and [Electron Forge](https://electronforge.io/). 

1. Install Node with one of the pre-built installers available from the Node website. 

After installing Node, Electron and Electron Forge are installed using `npm` from the command line. On Windows, Node installs a Node.js command prompt that I used for any `npm` and `electron-forge` commands (rather than the default Windows command prompt). The Node installer provides Windows users with the option to automatically install the necessary tools, including [Chocolatey](https://chocolatey.org/). I checked the box to automatically install these tools, but I'm not sure if this step is necessary to build the Electron app. 

2. Run `npm install -g electron-forge` in the Terminal (macOS) [[1]](#1) or Node.js command prompt (Windows) to install Electron Forge globally. 

3. Clone or download the GitHub repository for the RSE template.

Edit: The RSE template hasn't been updated in over a year and the Node ecosystem evolves rapidly. I [forked the repository](https://github.com/hinkelman/r-shiny-electron), searched for each dependency package in `package.json` via the [npm website](https://www.npmjs.com), and updated version numbers to the latest version found in the npm package search. I also removed `electron-forge` from the devDependencies because it is installed globally. If you are having trouble with the original repository, you should try the forked version of the RSE template repository.

### Build process on macOS

The RSE template repository includes a simple Shiny app. As a first step, I recommend following the steps below to confirm that you can build the executable with the simple app before trying your own app. At the end of this post, I will write more about extra steps required to build the executable for the DSM2 Viz Tool. 

1. Open the Terminal and change the directory to the location of `r-shiny-electron` or `r-shiny-electron-master` (see [how to use the Terminal](https://macpaw.com/how-to/use-terminal-on-mac)).

2. Run `npm install --save-dev electron` in the Terminal to install Electron locally. 

3. Run `npm install` to set up the project in the `r-shiny-electron` directory.

You will get lots of warnings about outdated packages [[2]](#2). I tried to fix the warnings with `npm audit fix`, but ran into problems and started over with a fresh clone of the RSE template.

4. Download R binary for macOS by running `./get-r-mac.sh` [[3]](#3).

5. Identify and download packages used in Shiny app by running `Rscript add-cran-binary-pkgs.R`.

6. Run `npm start` to test if app launches (and works correctly).

6. If the app works correctly, run `electron-forge make` to build the macOS executable.

The build process will create a folder called `Out` in the `r-shiny-electron` directory. In `Out/r-shiny-electron-darwin-x64`, you will have an executable (`r-shiny-electron.app`) that you can run to test that the app is working correctly. In `Out/make`, you will have a zip file that contains the executable but is better for distributing because of the smaller file size [[4]](#4).

It is possible to [build a Windows installer on macOS](https://github.com/dirkschumacher/r-shiny-electron/issues/25), but it "takes forever." I tried going down this road, but bailed after the process had run for 2 hours without finishing because I had access to a Windows machine for building the app.

### Build process on Windows

As mentioned in the macOS section, I recommend following the steps below to confirm that you can build the executable with the simple Shiny app available through the RSE template before trying your own app.

1. Install [Cygwin](https://cygwin.com/) and the [wget package](https://superuser.com/questions/693284/wget-command-not-working-in-cygwin), which is not installed by default. The `wget` function is used in `./get-r-win.sh`. If you have `wget` from another installation process, then you might be able to skip this step. 

2. Install `innoextract` with Chocolatey by running `choco install innoextract` from the command prompt with administrative privileges, i.e., right click on the command prompt and choose `Run as administrator`. `innoextract 1.8` is required. If you have a previous version of innoextract installed, then run `choco upgrade innoextract` [[5]](#5).

3. Open the Node.js command prompt and change the directory to the location of `r-shiny-electron` or `r-shiny-electron-master`.

4. Run `npm install --save-dev electron` in the Node.js command prompt to install Electron locally. 

5. Run `npm install` in the Node.js command prompt to set up the project in the `r-shiny-electron` directory.

6. Change the directory to `r-shiny-electron` in Cygwin, e.g., type `cd` in the Cygwin terminal and drag the `r-shiny-electron` folder to Cygwin to paste the path. Use [Notepad++](https://notepad-plus-plus.org) to open `./get-r-win.sh` and confirm that the [EOL characters are in Unix format](https://learningintheopen.org/2013/03/07/microsoft-windows-cygwin-error-r-command-not-found/). Download R binary for Windows by running `./get-r-win.sh` in Cygwin.

7. Identify and download packages used in the Shiny app by running `Rscript add-cran-binary-pkgs.R` from the [RStudio Terminal](https://support.rstudio.com/hc/en-us/articles/115010737148-Using-the-RStudio-Terminal). Remember to first change the directory to `r-shiny-electron`. You may also need to install the `automagic` package first.

8. Run `npm start` in the Node.js command prompt to test if the app launches (and works correctly) [[6]](#6).

9. If the app works correctly, run `electron-forge make` in the Node.js command prompt to build the Windows executable.

The build process will create a folder called `Out` in the `r-shiny-electron` directory. In `Out/make/squirrel.windows/x64`, you will have an installer executable called `r-shiny-electron-1.0.0 Setup.exe`.

### Customizing build process for DSM2 Viz Tool

After working through the build process on the r-shiny-electron example app, I downloaded a fresh version of the RSE template, renamed the `r-shiny-electron` directory to `DSM2-Viz-Tool`, removed the `app.R` file from `DSM2-Viz-Tool/shiny`, and replaced it with the files for the [DSM2 Viz Tool Shiny app](https://github.com/fishsciences/DSM2-Viz-Tool/tree/master/shiny). I also opened `package.json` in a text editor and changed a [few fields](https://github.com/fishsciences/DSM2-Viz-Tool/blob/master/package.json) (e.g., name, productName, version, description, author). 

Next, I started working through the build process described above. However, before running `npm start` I needed to add the binary files for the `rhdf5` and `Rhdf5lib` packages because `Rscript add-cran-binary-pkgs.R` doesn't find Bioconductor packages. You can find where R packages are installed on your machine by running `.libPaths()` from the R console. I manually copied and pasted the `rhdf5` and `Rhdf5lib` folders from the location indicated by the output of `.libPaths()` to `DSM2-Viz-Tool/r-mac/library` and `DSM2-Viz-Tool/r-win/library` on macOS and Windows, respectively. Then, I was able to finish the build process.

Working through this process involved plenty of frustration (because of my knowledge gaps), but it was extremely satisfying to arrive at the end goal of packaging the DSM2 Viz Tool as a standalone desktop application.

***

<a name="1"></a> [1] On macOS, you may need to provide permission to access `/usr/local/lib/node_modules`. [Solve the permission error](https://flaviocopes.com/npm-fix-missing-write-access-error/) by running `sudo chown -R $USER /usr/local/lib/node_modules` from the Terminal.

<a name="2"></a> [2] I personally find all the security warnings alarming, but [this article](https://www.voitanos.io/blog/don-t-be-alarmed-by-vulnerabilities-after-running-npm-install) suggests that I shouldn't worry about it.

<a name="3"></a> [3] The R version specified in this shell script should be the same R version that was used to build the shiny app.

<a name="4"></a> [4] On macOS Catalina, the first time that you run the app, you will be prompted to give permmission to access the file system. After granting permission, you may need to quit the app and then launch it again.

<a name="5"></a> [5] If that doesn't work, run `choco uninstall innoextract` and then run `choco install innoextract`.

<a name="6"></a> [6] One of the requirements is that `git` is installed. An installer is available [here](https://git-scm.com/download/win).

