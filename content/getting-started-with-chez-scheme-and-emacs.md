+++
title = "Getting started with Chez Scheme and Emacs on macOS and Windows"
date = 2019-08-03
[taxonomies]
categories = ["Chez Scheme", "Emacs"]
tags = ["REPL", "packages", "libraries", "extensions", "Raven", "geiser"]
+++

I [recently decided](/post/exploring-scheme-implementations/) to switch my attention from learning [Racket](https://racket-lang.org) to [Chez Scheme](https://cisco.github.io/ChezScheme/). One of the reasons that I [chose Racket](/post/programming-horizons/) was because of how easy it is to get up and running. Setting up a development environment for Chez requires jumping through a few more hoops. In this post, I document those hoops. Disclaimer: The suggestions in this post may not represent best practice. I will update the post as I become more experienced with Chez and Emacs.

<!-- more -->

***

## Chez Scheme

### Installation

On macOS, I installed Chez with [Homebrew](https://brew.sh).

```
$ brew install chezscheme
```

On Windows [[1]](#1), I used the [Windows installer](https://cisco.github.io/ChezScheme/#get).

So far, so good. Nothing tricky about installing Chez Scheme.

### REPL

To launch the Chez [REPL](https://en.wikipedia.org/wiki/Read–eval–print_loop) on macOS, open Terminal and type `chez` [[2]](#2). On Windows, four Chez programs were installed (32- or 64-bit threaded and unthreaded). Open any of those programs to get a Chez REPL [[3]](#3). 

Test the REPL with simple expression.

```
> (+ 100 10 1)
111
```

The REPL has several nice features including:

* Navigate through previous expressions with the up and down arrow keys.
* Autocomplete functions and paths with <kbd>TAB</kbd>.
* Write and edit multi-line expressions.

```
> (define (example x y z)
    (if (> x 0)
        (+ y z)
        (- y z)))
> (example 1 2 3)
5
```
When navigating through previous expressions, only the first line of a multi-line expression is shown. To see (and edit) all lines, type <kbd>CTRL</kbd>+<kbd>L</kbd>. In the middle of an expression, <kbd>RET</kbd> creates a new line; to enter an expression from the middle of an expression, use <kbd>CTRL</kbd>+<kbd>J</kbd>.

### Library Directory

Chez does not come with a package manager, but there are 3rd-party options, e.g., [Akku](https://akkuscm.org). In this post, though, I will describe manual package management. 

`library-directories` returns the directories where Chez looks for libraries. 

```
> (library-directories)
(("." . "."))
```

The `"."` indicates that Chez is looking in the current directory [[4]](#4). If you are using a project-based workflow, then you could include your dependencies in the current directory, perhaps in a `lib` folder. For a 'global' approach, I created a library directory at `/Users/username/scheme/lib` on macOS, and at `C:\scheme\lib` on Windows.

Before we go over where to stash that directory information, let's cover library extensions.

```
> (library-extensions)
((".chezscheme.sls" . ".chezscheme.so") (".ss" . ".so")
  (".sls" . ".so") (".scm" . ".so") (".sch" . ".so"))
```

These are the file extensions that Chez uses when searching the library directories.

On macOS [[5]](#5), you edit `.bash_profile` to add information on library directories and extensions. From a Terminal window, open `.bash_profile` with the [Nano text editor](https://www.nano-editor.org). 

```
$ nano .bash_profile
```

These lines add a new directory to `library-directories` and a new extension to `library-extensions`.

```
export CHEZSCHEMELIBDIRS="/Users/username/scheme/lib:"
export CHEZSCHEMELIBEXTS=".sc::.so:"
```

The `:` at the end is used to indicate that the new entries should be appended to the existing entries. Remove the `:` to replace the default values with the new entries. After saving `.bash_profile`, enter the following command in the Terminal.

```
$ source .bash_profile
```

Now, from a Chez REPL, we can see the effect of our changes.

```
> (library-directories)
(("/Users/username/scheme/lib" . "/Users/username/scheme/lib")
  ("." . "."))
> (library-extensions)
((".sc" . ".so") (".chezscheme.sls" . ".chezscheme.so") (".ss" . ".so")
  (".sls" . ".so") (".scm" . ".so") (".sch" . ".so"))
```

If we have a library at `/Users/username/scheme/lib/srfi/s1/lists.sls`, then we import the library with `(import (srfi s1 lists))`, i.e., you pass the components of the path to import. If you can't import the library, look at the `library` call at the top of `lists.sls`, for example, because that will give you a clue of where the library expects to be placed in `library-directories`. 

```
> (xcons 1 2)
Exception: variable xcons is not bound
Type (debug) to enter the debugger.
> (import (srfi s1 lists))
> (xcons 1 2)
(2 . 1)
```

On Windows 10, type `env` in the search box in the task bar and open the program to `Edit the system environment variables`. Then click the `Environment Variables` button. Click the button to create a new system variable. Type `CHEZSCHEMELIBDIRS` and `C:\scheme\lib;` in the name and value fields, respectively. Click `OK`. Repeat the process using `CHEZSCHEMELIBEXTS` and `.sc;;.so;` in the name and value fields. The `;` in the values fields has the same meaning as the `:` on macOS.

***

## Emacs

I'm not aware of an [IDE](https://en.wikipedia.org/wiki/Integrated_development_environment) for Chez Scheme, but pairing a good text editor with the Chez REPL provides a decent development environment. I chose [Emacs](https://www.gnu.org/software/emacs/emacs.html) as a text editor. I'm slowly becoming more comfortable with Emacs, but I'm far from proficient.

### Installation

On macOS, I installed Emacs with [Homebrew](https://brew.sh).

```
$ brew cask install emacs
```

On Windows, I installed [MSYS2](https://www.msys2.org) and then ran the following command from within MSYS2 to install Emacs.

```
$ pacman -S mingw-w64-x86_64-emacs
```

On macOS, I open Emacs via the icon in my applications folder. On Windows, I launch Emacs by typing `emacs` in the MSYS2 console. 

### Basic Usage

The power of Emacs is in the keyboard shortcuts and customization. I'm too early in my journey to have unlocked much of that potential. When you are browsing info on Emacs, you will see shorthand for referring to keyboard combinations, e.g., `C-x` `C-f` corresponds to <kbd>CTRL</kbd>+<kbd>X</kbd> followed by <kbd>CTRL</kbd>+<kbd>F</kbd>. The other important key is the meta key with `M` as the shorthand. On my MacBook, the meta key is <kbd>option</kbd>. On Windows, the default meta key is <kbd>ALT</kbd>. Similar to `.bash_profile`, Emacs can be customized through commands saved in the `.emacs` file [[6]](#6).

### Geiser

[Geiser](https://www.nongnu.org/geiser/) is a package that provides the ability to run several different Scheme implementations from within Emacs. We can install Geiser through [MELPA](https://melpa.org/#/). 

Open Emacs, enter `C-x` `C-f` to find a file, and type `.emacs` at the prompt. On macOS, I added the following to `.emacs`. 

```
(require 'package)
;;; either the stable version:

(add-to-list 'package-archives
  ;; choose either the stable or the latest git version:
  '("melpa-stable" . "https://stable.melpa.org/packages/"))
  ;; '("melpa-unstable" . "https://melpa.org/packages/"))

(package-initialize)
```

On Windows, I added the more extensive code provided on [MELPA's Getting Started page](https://melpa.org/#/getting-started). 

Save `.emacs` and restart Emacs. Then type `M-x` followed by `package-refresh-contents`. If that is successful, you will see the message `Package refresh done` in the [minibuffer](https://www.gnu.org/software/emacs/manual/html_node/emacs/Minibuffer.html). To install Geiser, type `M-x` and then `package-install`. In response to the `Install package: ` prompt, type `geiser` and hit return.

To customize Geiser, I used the menu options rather than directly editing the `.emacs` file. Choose `Options/Customize Emacs/Specific Group...` and type `geiser` at the prompt. Click on `Geiser Chez` and change the location of the binary. 

On mac0S...

```
/usr/local/Cellar/chezscheme/9.5.2/bin/chez
```

On Windows..

```
C:\Program Files\Chez Scheme 9.5\bin\ta6nt\scheme
```

Click `Apply and Save`. Click the `Geiser` link next to `Parent groups:` Click on `Geiser Implementation` and change the default implementation to `chez`. Click `Apply and Save`. Restart Emacs.
  
The Chez REPL is launched through Emacs with `M-x` followed by `run-chez`. You can navigate through the previous expressions with <kbd>ESC</kbd>+<kbd>P</kbd> and <kbd>ESC</kbd>+<kbd>N</kbd> [[7]](#7). Multi-line expressions, autocomplete, and syntax highlighting are also supported. 

### Library Directory

Apparently, the changes that we made to `.bash_profile` on macOS and the environment variables on Windows to point Chez to libraries and extensions are not picked up by the Chez REPL as used by Geiser. We need to add a couple of lines to `.emacs`.

On macOS...

```
(setenv "CHEZSCHEMELIBDIRS" "/Users/username/scheme/lib:")
(setenv "CHEZSCHEMELIBEXTS" ".sc::.so:")
```

On Windows...

```
(setenv "CHEZSCHEMELIBDIRS" "C:\\scheme\\lib;")
(setenv "CHEZSCHEMELIBEXTS" ".sc;;.so;")
```

***

UPDATE (2019-08-20): In Emacs, I eventually noticed that there is an option to highlight matching parantheses, which I find very helpful. Select `Options/Highlight Matching Parantheses` and then `Options/Save Options`. I've also started using [company-mode](http://company-mode.github.io) for text completion. I was also pleased to discover that reindenting lines in Emacs is as simple as selecting the section to indent and pressing <kbd>TAB</kbd>.

UPDATE (2019-12-04): Add the following lines to your `.emacs` file for `scheme-mode` to recognize the `.sc` and `.sls` file extensions that are used with scheme code.

```
(add-to-list 'auto-mode-alist
             '("\\.sls\\'" . scheme-mode)
             '("\\.sc\\'" . scheme-mode))
```

UPDATE (2019-12-13): In addition to using <kbd>TAB</kbd> to reindent lines in Emacs, my other most used keyboard shortcuts are for executing, commenting, and selecting code. To execute the code in an s-expression, place your cursor at the end of the s-expression and type `C-x` `C-e` [[8]](#8). If the executed code displays any output, it will be shown in the minibuffer and not the REPL [[9]](#9). To evaluate several s-expressions, highlight the region and type `C-c` `C-r`. To select an s-expression, place your cursor at the beginning of the s-expression and type `M-C-space` (where space is the space bar). I've done a lot of fumbling around trying to select s-expressions by dragging the cursor with the mouse so I'm excited to recently discover this last one.

***

<a name="1"></a> [1] My experience with Chez Scheme on Windows is limited to installation and set up as described in this post. But I'm under the impression that working with Chez Scheme on Windows involves more friction than on macOS or Linux.

<a name="2"></a> [2] You can also type `petite` to launch Petite Chez Scheme. See [here](https://cisco.github.io/ChezScheme/csug9.5/use.html) for more information on the differences between Chez Scheme and Petite Chez Scheme.

<a name="3"></a> [3] Alternatively, add one of the four versions of Chez Scheme to the Windows path environment variable by typing `env` in the search box in the task bar and then opening the program to `Edit the system environment variables`. Then click the `Environment Variables` button. Select the row for `Path` and then click on the `Edit...` button. Then click the `New` button and add the path. For the 64-bit threaded version, use `C:\Program Files\Chez Scheme 9.5\bin\ta6nt`. Now, you can launch Chez Scheme from the command prompt with `start scheme` and run Scheme programs with `start scheme path\to\myschemefile.ss`.

<a name="4"></a> [4] You can find the current directory by running `current-directory`.

<a name="5"></a> [5] If you are using macOS Catalina, the default shell is Zsh, not Bash. You simply need to replace `.bash_profile` with `.zshenv` in the instructions in this post.

<a name="6"></a> [6] More complicated file structures for customizing Emacs are possible, but my proficiency with Emacs is not at that level, yet.

<a name="7"></a> [7] Because my MacBook doesn't have a physical <kbd>ESC</kbd>, I prefer `C-[` as an alternative to <kbd>ESC</kbd> and, thus, navigate among previously entered expressions with `C-[` `p` and `C-[` `n`. Alternatively, you could use [<kbd>CAPS LOCK</kbd> as <kbd>ESC</kbd>](https://stackoverflow.com/a/40254864).

<a name="8"></a> [8] If you are receiving the message, "No Geiser REPL for this buffer", then Geiser is [struggling to figure out which Scheme implementation to use](http://geiser.nongnu.org/geiser_4.html#The-source-and-the-REPL). One solution is to delete all of the non-Chez implementations by choosing `Options/Customize Emacs/Specific Group...`, typing `geiser` at the prompt, selecting `Geiser Implementation`, and deleting all the non-Chez implementations under `Geiser Active Implementations`. Click `Apply and Save`. Restart Emacs.

<a name="9"></a> [9] If you want the output of the code displayed in the REPL, you will have to copy and paste it to the REPL (AFAIK).