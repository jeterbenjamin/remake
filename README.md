# remake

Forked in case the original is removed

[![Build Status](https://travis-ci.org/richfitz/remake.svg?branch=master)](https://travis-ci.org/richfitz/remake)
[![Build status](https://ci.appveyor.com/api/projects/status/yltv2lpn046a7e93/branch/master?svg=true)](https://ci.appveyor.com/project/richfitz/remake/branch/master)
[![Coverage Status](https://coveralls.io/repos/richfitz/remake/badge.svg?branch=master)](https://coveralls.io/r/richfitz/remake?branch=master)

Make-like build management, reimagined for R.

See below for [installation instructions](#installation).

# The idea

"[make](http://en.wikipedia.org/wiki/Make_(software))",
when it works, is wonderful.  Being able to change part of a complicated system and the re-make, updating only the parts of the system that have changed is great.  While it gets some use It's very heavily tailored towards building software though.  While make can be used to create reproducible research workflows (e.g. [here](http://www.bioinformaticszen.com/post/decomplected-workflows-makefiles/) and [here](http://kbroman.org/minimal_make/)), it is a challenge.

The idea here is to re-imagine a set of ideas from make but built for R.  Rather than having a series of calls to different instances of R (as happens if you run make on R scripts), the idea is to define pieces of a pipeline within an R session.  Rather than being language agnostic (like make must be), `remake` is unapologetically R focussed.

**Note**: This package is under heavy development (as of May 2015), so things may change under you if you start using this now.  However, the core format seems to be working on some nontrivial cases that we are using in our own work.  At the same time, if you're willing to have things change around a bit feel free to start using this and post [issues](https://github.com/richfitz/remake/issues) with problems/friction/ideas etc and the package will reflect your workflow more.

**Note**: Between versions `0.1` and `0.2.0` the database format has changed.  This will require rebuilding your project.  This corresponds to adding the dependency on [`storr`](https://github.com/richfitz/storr).  Everything else should remain unchanged though.

## What remake does

You describe the beginning, intermediate and end points of your analysis, and how they flow together.

* "**targets**" are the points in your analysis.  They can be either files (data files for input; tables, plots, knitr reports for output) or they can be R objects (representing processed data, results, fitted models, etc).
* "**rules**" are how the targets in your analysis relate together and are simply the names of R functions.
* "**dependencies**" are the targets that need to already be made before a particular target can run (for example, a processed data set might depend on downloading a file; a plot might depend on a processed data set).

There might be very few steps or very many, but `remake` will take care of stepping through the analysis in a correct order (there can be more than one correct order!).

## Example

Here's a very simple analysis pipeline that illustrates the basic idea:

1. Download some data from the web into a local file
2. Process that file to prepare it for analysis
3. Create a plot from that file
4. Create a knitr report that uses the same set of objects

The remakefile that describes this pipline might look like this:

```yaml
sources:
  - code.R

targets:
  all:
    depends: plot.pdf

  data.csv:
    command: download_data(target_name)

  processed:
    command: process_data("data.csv")

  plot.pdf:
    command: myplot(processed)
    plot: true

  report.md:
    depends: processed
    knitr: true
```

(this is a [yaml](http://yaml.org) file).  The full version of this file, with explanations, is [here](doc/remake.yml).

You still need to write functions that carry out each step; that might look something like [this](doc/code.R), but it would define the functions `download_data`, `processs_data` and `myplot`.  Remake can then be run from within R:

```r
remake::make()
# [ BUILD ] data.csv            |  download_data("data.csv")
# [ BUILD ] processed           |  processed <- process_data("data.csv")
# [ BUILD ] plot.pdf            |  myplot(processed) # ==> plot.pdf
# [       ] all
```

The "`BUILD`": next to each target indicates that it is being run (which may take some time for a complicated step) and after the pipe a call is printed that indicates what is happening (this is a small white lie).

Rerunning remake:

```r
remake::make()
# [    OK ] data.csv
# [    OK ] processed
# [    OK ] plot.pdf
# [       ] all
```

Everything is up to date, so remake just skips over things.

There are also special [`knitr`](http://yihui.name/knitr) targets:

```r
remake::make("report.md")
# [    OK ] data.csv
# [    OK ] processed
# [       ] report.Rmd
# [  KNIT ] report.md            |  knitr::knit("report.Rmd", "report.md")
```

This arranges for the target `processed`, on which this depends (see [the remakefile](doc/remake.yml)) to be passed through to `knitr`, along with all the functions defined in `code.R`, and builds the report `report.md` from the knitr source `report.Rmd` (the source is [here](doc/report.Rmd)).  Note that because `processed` was already up to date, `remake` skips rebuilding it.

`remake` can also be run from the command line (outside of R), to make it easy to include as part of a bigger pipeline, perhaps using make! (I do this in my own use of remake).

Rather than require that you buy in to some all-singing, all-dancing workflow tool, `remake` tries to be agnostic about how you work: there are no special functions within your code that you need to use.  You can also create a linear version of your analysis at any time:

```r
remake::make_script()
# source("code.R")
# download_data("data.csv")
# processed <- process_data("data.csv")
# pdf("plot.pdf")
# myplot(processed)
# dev.off()
```

## Other core features:

* `remake` determines if any dependencies have changed when need running your analysis.  So if a downloaded data file changes, everything that depends on it will be rebuilt when needed.
  - however, rather than rely on file modification times, `remake` uses a hash (like a digital fingerprint) of the file or object to determine if the contents have *really* changed.  So inconsequential changes will be ignored.
* Object targets are persistent across sessions, so manual saving of objects is not necessary.  This avoids a lot of manual caching of results that tends to litter long-running analysis code.
* `remake` also checks if the *functions* used as rules (or called from those functions) have changed and will rebuild if these have changed (for the rationale here, see [here](doc/reproducible_research.md)).
* Because `remake` keeps track of which files and objects it created, it can automatically clean up after itself.  This makes it easy to rerun the entire analysis beginning-to-end.
  - three levels of cleaning (tidy, clean and purge) are provided to categorise how much you want to keep a particular target.
* Support for easily making figures and running [`knitr`](http://yihui.name/knitr) as special targets.
* Automate installation of dependencies.
* Automate curation of `.gitignore` files to prevent accidentally committing large output to your repository.
* (Very early) support for archiving a set of analyses that other users can import.
  - This means you can share results of long-running analyses/simulations and people can easily run the last steps of your analyses.
  - Eventually this will interface with [rfigshare](https://github.com/ropensci/rfigshare) and GitHub [releases](https://github.com/blog/1547-release-your-software).

## Real-world examples

* A large analysis of [adequacy of models used in comparative phylogenetics](https://github.com/richfitz/modeladequacy/blob/master/analysis/remake.yml).
* [How much of the world is woody?](https://github.com/richfitz/wood_remake) (this is a port of the make-based [original](https://github.com/richfitz/wood))
* A large [data synthesis project](https://github.com/dfalster/baad) (unfortunately complicated by being a templated remakefile).
* The accounts for a [coffee coop](https://github.com/aammd/CoffeeCoop)

## Tutorials

Some tutorials on using remake with different datasets. 

* [Tutorial on remake from Ecostats workshop](https://github.com/nicercode/2015.12.08-EcoStats)
* [Tutorial on remake using the Gapminder data](https://github.com/ropenscilabs/remake-tutorial)
* [Tutorial on remake using cooking recipes as examples](https://github.com/krlmlr/remake-tutorial)


# Installation

Install using [devtools](https://github.com/hadley/devtools):

```r
devtools::install_github("richfitz/remake")
```

If you don't have devtools installed you will see an error "there is no package called 'devtools'"; if that happens install devtools with `install.packages("devtools")`.

remake depends on several R packages, all of which can be installed from CRAN.  The required packages are:

* [`R6`](http://cran.r-project.org/web/packages/R6) for holding things together
* [`yaml`](http://cran.r-project.org/web/packages/yaml) for reading the configuration
* [`digest`](http://cran.r-project.org/web/packages/digest) for efficiently hashing objects
* [`crayon`](https://github.com/gaborcsardi/crayon) for coloured output on the terminal (not in Rstudio or Rgui)
* [`optparse`](http://cran.r-project.org/web/packages/optparse) for a command line version (run from outside of an R session)

```r
install.packages(c("R6", "yaml", "digest", "crayon", "optparse"))
```

We also depend on `storr` for object storage:

```r
devtools::install_github("richfitz/storr")
```
