+++
title = "Getting started with Akku package manager for Scheme"
date = 2020-05-16
[taxonomies]
categories = ["Chez Scheme", "Akku"]
tags = ["libraries", "packages"]
+++

[Akku](https://akkuscm.org) is a package manager for Scheme that currently supports numerous R6RS and R7RS Scheme implementations [[1]](#1). I was slow to embrace Akku because I encountered some initial friction with installation and setup. Moreover, coming from R, I was more familiar with a global package management model than Akku's project-based workflow. In the meantime, I was content to manually manage the few libraries that I had downloaded from different repos and placed in a directory found by Chez's `(library-directories)`.

<!-- more -->

Recently, though, I spent part of a weekend playing with [Janet](https://janet-lang.org) and was seriously considering shifting my attention from Chez to Janet. At the end of the weekend, when I was taking stock of why I found Janet so appealing, it boiled down to lisp-like syntax with modern affordances and a built-in package manager. I decided that convenient syntax didn't trump Chez's solidity and stability, which left the package manager as the primary attraction. I came away from that experience with a renewed appreciation for Chez Scheme and resolved to push past my previous pain points and get up and running with Akku. I wasn't entirely successful. On Windows, I wasn't even able to install Akku. On macOS, I'm able to install and use Akku, but not publish packages. On Linux, Akku works seamlessly. 

In this post, I will walk through the basics of setting up Akku, setting up projects, and publishing packages.

### Setting up Akku

First, download the [Akku source version for Chez Scheme](https://github.com/weinholt/akku/releases). Then change your directory to the download location and run the following commands:

```
$ tar -xf akku-1.0.1.src.tar.xz
$ cd akku-1.0.1.src
$ ./install.sh
```

For some reason, this doesn't work with the latest version (9.5.4) of Chez Scheme (if you've installed Chez Scheme from source). If you are on Linux, you can use the pre-built version of Akku.

```
$ tar -xf akku-1.0.1.amd64-linux.tar.xz 
$ cd akku-1.0.1.amd64-linux/
$ ./install.sh
```

If neither the Chez version nor the pre-built version work for you, then you will need to install the release tarball (`akku-1.0.1.tar.gz`), which requires [Guile](https://www.gnu.org/software/guile/).

Add one of the following lines to `.bashrc`,  `.bash_profile`, `.zshenv`, or wherever you keep your shell configuration commands. Make sure to replace `username` with the your actual username.

On macOS...

```
export PATH="/Users/username/.local/bin:$PATH"
```

On Linux...

```
export PATH="/home/username/.local/bin:$PATH"
```

### Setting up a project

If you are starting a new project, the command `akku init` will initialize "a new project with a simple template that demonstrates a program, a library and a test."

```
akku init example-project
```

You should manually update the version, synopsis, authors, and license in `Akku.manifest`. If you are using Akku to install packages, then you will not need to manually update the depends field. However, Akku provides the option to [include direct dependencies](https://gitlab.com/akkuscm/akku/-/wikis/Direct-dependencies) from Git repositories, local directories, and URLs, which does require manually updating the depends field. If you using direct dependencies, be advised... 

> Packages with direct dependencies will not be accepted into the Akku.scm archive. The archive is a self-sufficient set of packages.

It is not necessary to first use `akku init`. You can start using Akku in an existing project. Let's pretend like we have an existing project.

```
$ mkdir existing-project
$ cd existing-project
$ akku update
$ akku install
```

`akku update` updates the local package index. It is a global command, not specific to a project. If you are having trouble installing a package, this is the first thing to try. `akku install` creates the hidden `.akku` directory [[2]](#2). When you install packages with `akku install pkgname`, the library files are placed in `.akku/lib` and the full source code is installed in `.akku/src`.

Let's install my `chez-stats` library.

```
akku install chez-stats
```

Now, if you take in a peak in `.akku/lib`, you will find `chez-stats` and the `srfi` library. `chez-stats` only has a dependency on `srfi` for testing [[3]](#3).

Installing `chez-stats` created the following `Akku.manifest` file, which indicates a dependency on the current version of `chez-stats`.

```
#!r6rs ; -*- mode: scheme; coding: utf-8 -*-
(import (akku format manifest))

(akku-package ("existing-project" "0.0.0-alpha.0")
  (synopsis "I did not edit Akku.manifest")
  (authors "Guy Q. Schemer")
  (license "NOASSERTION")
  (depends ("chez-stats" "^0.1.0"))
)
```

Let's now illustrate how `.akku/env` sets the environment to find the installed libraries. If we launch Chez (with `chez` or `scheme`) from the `existing-project` directory, and try to load `chez-stats`, we will be out of luck.

```
> (import (chez-stats))
Exception: library (chez-stats) not found
```

Calling `library-directories` illustrates the problem [[4]](#4).

```
> (library-directories)
(("/home/username/chez/lib"
    .
    "/home/username/chez/lib")
  ("." . "."))
```

If we load the Akku environment before launcing Chez,

```
$ .akku/env
$ scheme
```

then Chez knows where to look for the libraries used in this project.

```
> (library-directories)
(("/home/username/existing-project/.akku/lib"
   .
   "/home/username/existing-project/.akku/libobj"))
   
> (import (chez-stats))
> (mean (repeat 1e6 (lambda () (random-normal))))
0.0031616355735957662
```

I [accidentally discovered](https://gitlab.com/akkuscm/akku/-/issues/46) that if you are using `zsh` as your shell (new macOS default), and exporting a `CHEZSCHEMELIBDIRS` in `.zshenv`, then running `.akku/env` will not find the libraries installed in your package because `.zshenv` is apparently loaded after `.akku/env` (instead of when Terminal is opened). One fix is to add the following conditional logic to `.zshenv`. UPDATE (2020-11-16): I don't know if there was a recent change to Ubuntu, but I now also have this same problem using `bash` and have added these same lines to `.bashrc`.

```
if [[ -v AKKU_ENV ]]; then
    echo "AKKU_ENV is set"
else
    export CHEZSCHEMELIBDIRS="/Users/users/chez/lib:"
fi
```

So far all of the examples assume that you are working from the Terminal, but Akku also provides examples of `.chez-geiser` files that allow for [integration with Emacs and Geiser](https://gitlab.com/akkuscm/akku/-/wikis/Integration-with-Emacs-and-Geiser). If Geiser is not picking up the `.chez-geiser` file, then you might need to update Geiser. 

### Publishing a package

In preparation for publishing a package, we need to prepare and publish a GnuPG key (if you don't already have one).

```
$ sudo apt install gnupg
```

Generate a key and answer the subsequent prompts.

```
$ gpg --generate-key
```

Publish your key to a public key server with the following command (where `keyid` is replaced with your new public key, a long string of numbers and letters).

```
$ gpg --keyserver pgp.mit.edu --send-keys keyid
```

When you tag your repo, it will prompt you for your `GnuPG` credentials. I'm not sure if the message is necessary, but the first time I tried to tag the repo (on macOS), git automatically opened `vi`. Including a message with `-m` will spare you from that fate.

```
$ git tag -s v0.1.0 -m "initial release"
```

Next you need to push the tagged release.

```
$ git push --tags
```

Because I use two-factor authentication for GitHub, I needed to set up a [personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) to use as the password when pushing the tags.

The last step is to publish with Akku.

```
$ akku publish
```

I can't speak for the average time for a library to appear in the Akku package list, but I published three packages this week and all were up within 24 hours.

***

<a name="1"></a> [1] Chez Scheme, Chibi Scheme, GNU Guile, Gauche Scheme, Ikarus Scheme, IronScheme, Larceny Scheme, Loko Scheme, Mosh Scheme, Racket (plt-r6rs), Sagittarius Scheme, Vicare Scheme, Ypsilon Scheme

<a name="2"></a> [2] To toggle the visibility of files on macOS, use the keyboard shortcut `shift+command+period`. On Linux, use `control+H`. 

<a name="3"></a> [3] You can run the tests for `chez-stats` from the Terminal by first running `.akku/env` and then `scheme .akku/src/chez-stats/tests/test-chez-stats.sps`.

<a name="4"></a> [4] `/home/username/chez/lib` is where I put libraries that I want to make globally available.






