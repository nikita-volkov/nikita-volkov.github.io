---
layout: post
title: "Tutorial: Profiling Cabal projects"
tags: [haskell, cabal, profiling, tutorial]
description: A tutorial on how to set up Cabal projects for profiling
comments: true
---

Profiling in Haskell can be an overwhelmingly confusing task. There's plenty of little things you need to set up right to be able to perform it. This post aims to organize this information.

## Summary

1. The project must be configured with profiling enabled.

1. Some specific options must be set when configuring the profiling executable in the ".cabal" file.

1. All the project's dependencies must be installed with profiling enabled.

## Configuring the project

Well, there's nothing tricky here. All you need to do is just pass the `--enable-library-profiling` and `--enable-executable-profiling` flags to the `configure` task. E.g., here is how you configure a project to support profiling as well as running tests and benchmarks:

{% highlight bash %}
cabal configure --enable-library-profiling --enable-executable-profiling --enable-tests --enable-benchmarks
{% endhighlight %}

## Configuring the executable

Here's an example of a typical setup I use for profiling:

{% highlight text %}
executable my-project-profiling
  main-is:
    Profiling.hs
  ghc-options:
    -O2
    -threaded
    -fprof-auto
    "-with-rtsopts=-N -p -s -h -i0.1"
  build-depends:
    base >= 4.5 && < 5
{% endhighlight %}

#### Explanation of the GHC options:

* `-O2` enables aggressive optimization.
* `-threaded` enables concurrency.
* `-fprof-auto` enables automatic cost-centre annotations for profiling. [Details here](http://www.haskell.org/ghc/docs/7.8.2/html/users_guide/prof-compiler-options.html).
* `-with-rtsopts` bakes in the runtime settings.

#### Explanation of the runtime settings:

* `-N` sets the number of available processors for the executable to be equal to the amount of processors on the running system. Otherwise it's just 1.
* `-p` generates a time and allocation profiling report in a ".prof" file. [Details here](http://www.haskell.org/ghc/docs/7.8.2/html/users_guide/prof-time-options.html).
* `-s` outputs a report on garbage collection.
* `-h` generates a standard report on memory usage. Which can then be used to produce a plot in a postscript file using the "hp2ps" utility. [Details here](http://www.haskell.org/ghc/docs/7.8.2/html/users_guide/prof-heap.html).
* `-i0.1` sets the sampling frequency of memory profiling to every tenth of a second.

## Installing the dependencies

This gets trickier. Chances are, most of your dependencies were installed without profiling support and you need to reinstall them. Haskell package management has never been much friendly and it will require you to either drop the whole local packages' DB and reinstall all the dependencies with profiling enabled, or manually reinstall them all one by one.

Fortunately, since the version 1.17 Cabal comes with a "sandboxes" feature, which maintains an isolated dependencies DB per each project. This is a workaround for numerous Haskell package management issues and is a way to go for us. So before continuing please make sure that you have a proper version of Cabal.

### Initialize the project's sandbox:

{% highlight bash %}
cabal sandbox init
{% endhighlight %}

### Install the dependencies with support for profiling:

{% highlight bash %}
cabal install --only-dependencies --enable-library-profiling
{% endhighlight %}

Alternatively you can add a "cabal.config" file to the root of your project with the following line:

{% highlight text %}
library-profiling: True
{% endhighlight %}

This will ensure that all the dependencies you install with this project will be with support for profiling. So now you can simply run the following:

{% highlight bash %}
cabal install --only-dependencies
{% endhighlight %}

## Running

With all things set up you can finally run your profiling configuration with the following:

{% highlight bash %}
cabal run my-project-profiling && hp2ps -e8in -c my-project-profiling.hp
{% endhighlight %}

This command is a two-parter:

* `cabal run my-project-profiling` executes our profiling configuration and generates all the reports.
* `hp2ps -e8in -c my-project-profiling.hp` converts the generated "my-project-profiling.hp" into a postscript graph.

Obviously you have to replace "my-project-profiling" with whatever the name you've specified in your cabal configuration.

After running, you'll get all the reports generated. All that will be left to do is to analyse them. You can find plenty of tutorials on analysing the data throughout the web.

Happy profiling!
