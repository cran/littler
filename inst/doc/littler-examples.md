<!--
%\VignetteEngine{simplermarkdown::mdweave_to_html}
%\VignetteIndexEntry{Littler Examples}
%\VignetteEncoding{UTF-8}
-->
---
title: "Littler Examples"
author: "Dirk Eddelbuettel"
date: "Originally written 2015-10-28; Updated 2017-12-10, 2018-05-24, 2019-01-06, 2019-06-09 and 2020-06-16"
css: "water.css"
---

## Overview

We show and discuss a few of the files included in the `inst/examples/` source directory of
[littler](http://dirk.eddelbuettel.com/code/littler.html) (which becomes the `examples/` directory 
once installed). In a few cases we remove comment lines to keep things more concise on this page.
We use `$` to denote a shell (_i.e._ terminal) prompt.

Note that some systems (such as macOS) cannot install littler as `r` (as lower-case and upper-case
are by default the same; not a great idea).  And for example the `zsh` has `r` as a builtin so you
would have to use `/usr/bin/r`.  See the [littler FAQ](littler-faq.html) for more.

## Simple Direct Command-line Use

[littler](http://dirk.eddelbuettel.com/code/littler.html) can be used
directly on the command-line just like, say, `bc`, easily consuming standard
input from a pipe: 

```bash
$ echo 'cat(pi^2,"\n")' | r
9.869604
```

Equivalently, commands that are to be evaluated can be given on
the command-line

```bash
$ r -e 'cat(pi^2, "\n")'
9.869604
```

But unlike bc(1), GNU R has a vast number of statistical
functions. For example, we can quickly compute a `summary()` and show
a stem-and-leaf plot for file sizes in a given directory via

```bash
$ ls -l /boot | awk 'BEGIN {print "size"} !/^total/ {print $5}' | \
     r -de "print(summary(X[,1])); stem(X[,1])"
```

which produces something like 

```
   Min. 1st Qu.  Median    Mean 3rd Qu.    Max.
     13     512  110100  486900  768400 4735000

  The decimal point is 6 digit(s) to the right of the |

   0 | 0000001122222279222
   2 | 79444
   4 | 71888
   6 | 
   8 | 
  10 | 
  12 | 
  14 | 8
  16 | 4
  18 | 
  20 | 333
```

(Note that some systems may not have `/boot` in which case you can try `/sbin` or another
directory.)

As we saw in the preceding example, the program can also be shortened like
using the new `-d` option which reads from stdin and assigns to a `data.frame` named `X`. 

And, last but not least, this (somewhat unwieldy) expression can be stored in
a helper script (where we now switch to using an explicit `readLines()` on `stdin`):

```r
#!/usr/bin/env r

fsizes <- as.integer(readLines(file("stdin")))
print(summary(fsizes))
stem(fsizes)
```

(where calling `#!/usr/bin/env` is a trick from Python which allows one
to forget whether r is installed in `/usr/bin/r`, `/usr/local/bin/r`,
`~/bin/r`, ...).


## install.r: Direct CRAN Installation

This is one of my favourite
[littler](http://dirk.eddelbuettel.com/code/littler.html) scripts which I use
frequently to install packages off [CRAN](https://cran.r-project.org).

```r
#!/usr/bin/env r

if (is.null(argv) | length(argv)<1) {
  cat("Usage: installr.r pkg1 [pkg2 pkg3 ...]\n")
  q()
}

## adjust as necessary, see help('download.packages')
repos <- "https://cran.rstudio.com" 

## this makes sense on Debian where no packages touch /usr/local
lib.loc <- "/usr/local/lib/R/site-library"

install.packages(argv, lib.loc, repos)
```

I invoke it all the time with one, two or more packages to install (or
reinstall).

```bash
$ install.r digest RcppCNPy
```

It conveniently installs all dependencies, and uses the chosen target
directory, all while keeping my R prompt (or prompts with multiple sessions)
free to do other things.  Also, if used with `options("Ncpu")` set, then
(remote CRAN) packages will be installed in parallel.

## install2.r: With Cmdline Parsing

Thanks to the fabulous [docopt](https://github.com/edwindj/docopt.R)
package, we also have a variant `install2.r` with optional settings of
repo and location. It was first installed with
[littler 0.2.1](http://dirk.eddelbuettel.com/blog/2014/10/19/).
The current (_i.e._, 0.3.3 as of this writing) version is more featureful and longer and included to keep this brief. Some usage examples are

```sh
$ install2.r -l /tmp/lib Rcpp BH                         # install into given library
$ install2.r -- --with-keep.source drat                  # keep the source
$ install2.r -- --data-compress=bzip2 stringdist         # prefer bz2 compression
```

Another _very_ useful option is `-n N` or `--ncpus N` which will parallelize
the installation across `N` processes.  This can be set automagically by
setting `options("Ncpus")`.

### Installing From Sources

Starting with version 0.2.2, `install.r` and `install2.r` now recognise
installable source files.  So one can also do this: 

```bash
$ install.r digest_0.6.8.tar.gz
```

and the local source file will the installed via a call to `R CMD INSTALL`.


## check.r: Simple Checker

A related use case is to check packages via `check.r`.  This script runs `R
CMD check`, but also installs package dependencies first as tests may have
dependencies not yet satisfied on the test machine.  It offers a number of options:

```bash
$ check.r  -h
Usage: check.r [-h] [-x] [--as-cran] [--repo REPO] [--install-deps] [--install-kitchen] [--deb-pkgs PKGS...] [--use-sudo] [--library LIB] [--setwd DIR] [TARGZ ...]

-a --as-cran          customization similar to CRAN's incoming [default: FALSE]
-r --repo REPO        repository to use, or NULL for file [default: https://cran.rstudio.com]
-i --install-deps     also install packages along with their dependencies [default: FALSE]
-k --install-kitchen  even install packages 'kitchen sink'-style up to suggests [default: FALSE]
-l --library LIB      when installing use this library [default: /usr/local/lib/R/site-library]
-s --setwd DIR        change to this directoru before undertaking the test [default: ]
-d --deb-pkgs PKGS    also install binary .deb packages with their dependencies [default: FALSE]
-u --use-sudo         use sudo when installing .deb packages [default: TRUE]
-h --help             show this help text
-x --usage            show help and short example usage
$
```

## rcc.r: R CMD check wrapper

The script `rcc.r` will also check a source tarball, and offers another set of options:

```bash
$ rcc.r -h
Usage: rcc.r [-h] [-x] [-c] [-f] [-q] [--args ARGS] [--libpath LIBP] [--repos REPO] [PATH...]

-c --as-cran          should '--as-cran' be added to ARGS [default: FALSE]
-a --args ARGS        additional arguments to be passed to 'R CMD CHECK' [default: ]
-l --libpath LIBP     additional library path to be used by 'R CMD CHECK' [default: ]
-r --repos REPO       additional repositories to be used by 'R CMD CHECK' [default: ]
-f --fast             should vignettes and manuals be skipped [default: FALSE]
-q --quiet            should 'rcmdcheck' be called qietly [default: FALSE]
-h --help             show this help text
-x --usage            show help and short example usage
$
```

## build.r: Building Packages

A related script is `build.r` which I often use inside a source repository to quickly build
a source tarball. Like `rcc.r`, it has a switch `-f` or `--fast` to omit building of vignettes.

```bash
$ build.r            # without argument works on current directory
$ build.r digest/    # with directory argument builds tar.gz from repo in directory
```


## installGithub.r: GitHub install

Installation directly from [GitHub](https://github.com) is also popular. Here is an example: 

```bash
$ installGithub.r RcppCore/RcppEigen 
```
Installing from github is supported via the following helper script:

```r
#!/usr/bin/env r
#
# A simple example to install one or more packages from GitHub
#
# Copyright (C) 2014 - 2015  Carl Boettiger and Dirk Eddelbuettel
#
# Released under GPL (>= 2)

## load docopt and remotes (or devtools) from CRAN
suppressMessages(library(docopt))       # we need docopt (>= 0.3) as on CRAN
suppressMessages(library(remotes))      # can use devtools as a fallback

## configuration for docopt
doc <- "Usage: installGithub.r [-h] [-d DEPS] REPOS...

-d --deps DEPS      Install suggested dependencies as well? [default: NA]
-h --help           show this help text

where REPOS... is one or more GitHub repositories.

Examples:
  installGithub.r RcppCore/RcppEigen                     

installGithub.r is part of littler which brings 'r' to the command-line.
See http://dirk.eddelbuettel.com/code/littler.html for more information.
"

## docopt parsing
opt <- docopt(doc)
if (opt$deps == "TRUE" || opt$deps == "FALSE") {
    opt$deps <- as.logical(opt$deps)
} else if (opt$deps == "NA") {
    opt$deps <- NA
}

invisible(sapply(opt$REPOS, function(r) install_github(r, dependencies = opt$deps)))
```

## update.r: CRAN package update

One of the scripts I use the most (interactively) is `update.r` which updates installed packages.
An earlier version looked like the following example, the current version is
a little longer and has more features:

```r
#!/usr/bin/env r 
#
# a simple example to update packages in /usr/local/lib/R/site-library
# parameters are easily adjustable

## adjust as necessary, see help('download.packages')
repos <- "https://cloud.r-project.org" 

## this makes sense on Debian where no package touch /usr/local
lib.loc <- "/usr/local/lib/R/site-library"

## r use requires non-interactive use
update.packages(repos=repos, ask=FALSE, lib.loc=lib.loc)

```

As above, it has my preferred mirror and library location hard-wired.

Another _very_ useful option is `-n N` or `--ncpus N` which will parallelize
the upgrade across `N` processes.  This can be set automagically by setting
`options(Ncpus)`.


## knit.r: Calling knitr

Here is another convenience script, `knit.r`,  which _knits_ a given file after
testing the file actually exists.

```r
#!/usr/bin/r
#
# Simple helper script for knitr
#
# Dirk Eddelbuettel, May 2013
#
# GPL-2 or later

if (is.null(argv)) {
    cat("Need an argument FILE.Rnw\n")
    q(status=-1)
}


file <- argv[1]
if (!file.exists(file)) {
    cat("File not found: ", file, "\n")
    q(status=-1)
}

require(knitr)
knit2pdf(file)

```


## roxy.r: Running roxygen

Similar to the previous example, the script `roxy.r` one uses roxygen to extract
documentation from R files -- either in the current directory, or in the
given directory or directories.

```r
#!/usr/bin/r
#
# Simple helper script for roxygen2::roxygenize() 
#
# Dirk Eddelbuettel, August 2013
#
# GPL-2 or later

## load roxygen
library(roxygen2)

## check all command-line arguments (if any are given) for directory status
argv <- Filter(function(x) file.info(x)$is.dir, argv)

## loop over all argument, with fallback of the current directory, and
## call compileAttributes() on the given directory
sapply(ifelse(length(argv) > 0, argv, "."), FUN=roxygenize, roclets="rd")

```

## compAttr.r: Compiling Attributes

The next script, `compAttr.`,  can be used with
[Rcpp](http://dirk.eddelbuettel.com/code/rcpp.html), and particularly is
powerful _Attributes_ feature, in order to auto-generate helper code. It is
similar to the preceding script, but invokes `compileAttributes()` instead.

```r
#!/usr/bin/r
#
# Simple helper script for compileAttributes() 
#
# Dirk Eddelbuettel, July 2014
#
# GPL-2 or later

## load Rcpp
suppressMessages(library(Rcpp))

## check all command-line arguments (if any are given) for directory status
argv <- Filter(function(x) file.info(x)$is.dir, argv)

## loop over all argument, with fallback of the current directory, and
## call compileAttributes() on the given directory
sapply(ifelse(length(argv) > 0, argv, "."), compileAttributes)

```

## render.r: Render Markdown

The `render.r` script generalizes the earlier `knit.r` to convert Markdown
into its designated output.

```r
#!/usr/bin/env r
#
# Another example to run a shiny app
#
# Copyright (C) 2016  Dirk Eddelbuettel
#
# Released under GPL (>= 2)

suppressMessages(library(docopt))       # we need docopt (>= 0.3) as on CRAN

## configuration for docopt
doc <- "Usage: render.r [-h] [-x] [FILES...]

-h --help            show this help text
-x --usage           show help and short example usage"

opt <- docopt(doc)			# docopt parsing

if (opt$usage) {
    cat(doc, "\n\n")
    cat("Examples:
  render.r foo.Rmd bar.Rmd        # convert two given files

render.r is part of littler which brings 'r' to the command-line.
See http://dirk.eddelbuettel.com/code/littler.html for more information.\n")
    q("no")
}

library(rmarkdown)

## helper function 
renderArg <- function(p) {
    if (!file.exists(p)) stop("No file '", p, "' found. Aborting.", call.=FALSE)
    render(p)
}

## render files using helper function 
sapply(opt$FILES, renderArg)

```



