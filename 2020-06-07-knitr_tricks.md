---
layout: post
title: "Knitr Tricks I Learned the Hard Way"
comments:  true
published:  true
author: "Zachary Burchill"
date: 2020-06-07 00:30:00
permalink: /knitr_tricks/
categories: [R,knitr,plotly,chunks,"interactive plots","data visualization","Jekyll","GitHub Pages"]
output:
  html_document:
    mathjax:  default
    fig_caption:  false
---






You would have thought that I would have learned to RTFM by now, but too often I find myself learning the subtleties of an R package by tearing it apart.  I swear that I read the documentation and do the requisite Googling and StackExchanging, but it always seems whatever I want to do is just a little too esoteric for the mainstream. I'm just too addicted to R's wonderful metaprogramming abilities, and I guess making that work often involves needing to understand deeper parts of the code.

I got distracted by these types of problems most recently with the `knitr` package in R.  `knitr` is used to make basically any type of document these days, and the package does an amazing job of walking the line between being user-friendly and deep and customizable. Here I'll show you a few tricks that will hopefully get you thinking about how *you* can customize knitr!

<!--more-->


## Fanfare when your document is done {#beep}

First off, I love the [`beepr` package](https://cran.r-project.org/web/packages/beepr/index.html), which basically just lets you play sounds from R.  As I'm writing my thesis, I have a few documents I've been knitting that can take a relatively long time to knit.  So that I don't sit around waiting for them to finish, I put something like


{% highlight r %}
beepr::beep(3)
{% endhighlight %}

in a chunk at the end of the document, so that right before it finishes loading it plays a trumpet sound, alerting me that it's done.  That way, I don't have to be working in that window to know that I can switch back.

Although adding a simple chunk to the end of each of my reports is hardly any work, it wasn't *elegant* enough for me.  I wanted to something *programmatic*.  Unfortunately, knitr doesn't have a way of generating new chunks programmatically within the document, but you *can* set a hook that will happen right at the end of the document[^1].

Voila: by calling `beep_on_knit()` in the your document, you'll automatically hear a sound when your document's done!


{% highlight r %}
#' Beep on successful knit
#'
#' This function requires the `beepr` package. 
#' It will play a sound when the document finishes knitting.
#'
#' @param beep_sound Input to `beepr::beep()` for what sound to play
#' @param sleep The amount of seconds to sleep before 
#' moving on (so that the sound isn't cut off after it knits)
#'
#' @export
beep_on_knit <- function(beep_sound=3, sleep=4) {
  library(beepr)
  # In case something else modified the document hook,
  #   we don't want to get rid of it
  prev_fn <- knitr::knit_hooks$get("document")
  knitr::knit_hooks$set(document = function(x) {
    beepr::beep(beep_sound)
    Sys.sleep(sleep)
    prev_fn(x)
  })
}
{% endhighlight %}

## Autolabeling figures in LaTeX

Almost all of the reports I'm making for my thesis are PDFs in LaTeX. My advisor is very insistent that each figure have a caption and is referenced by number in each report.  Currently, the way I've solved this is by labeling the figures in their captions, like so:


{% highlight text %}
```{r test_fig, fig.cap="This is the caption.\\label{fig:testfig}"}
<plotting code here>
```
{% endhighlight %}

Having to type `fig.cap="...\\label{fig:...}"` for every single figure has become super annoying. The docs seem to suggest that labels for figures are generated automatically ([see `fig.lp` under 'Plots'](https://yihui.org/knitr/options/)), but it doesn't do that in my RStudio/RMarkdown setup. I also felt I knew enough about the package at this point to whip up my own solution.

I made a custom option hook that would add `\\label{fig:X}` to the caption where `X` is a slightly edited/trimmed version of the label. No more extra typing!


{% highlight r %}
#' Autolabel figures in Latex
#'
#' I hate having to add `\\label{fig:blah}` to `fig.cap` 
#' in Latex .Rmd files.  There's probably a better way to do 
#' this, but if you call this function, it will automatically 
#' add `\\label{fig:<label>}` to the `fig.cap` option of any 
#' chunk with a `fig.cap` and a label, substituting whitespace 
#' stretches (after trimming it)
#'
#' @export
autolabel_latex_figs <- function() {
  knitr::opts_hooks$set(
    .autolabel =
      function(options) {
        if (isTrue(options$.autolabel) && !is.null(options$label)) {
          label = trimws(gsub("\\s+", "_", options$label))
          options$fig.cap = paste0(options$fig.cap,
                                   "\\label{fig:",label,"}")
          options
        } else {
          options
        }
      })
  knitr::opts_chunk$set(.autolabel = TRUE)
}
{% endhighlight %}


## Using Plotly and HTML widgets with Jekyll the right, _right_ way {#rightplotly}

So you may remember [this previous post of mine]({{ site.baseurl }}{% post_url 2020-04-04-plotly_with_jekyll %}) talking about how to make interactive plots with Jekyll and GitHub Pages.  While I still stand by the fact that I was *righter* than anything else online, I recently found out that most of that post was just reinventing the wheel.

In that post, I ended up making a stateful plotting function that collected the CSS/JavaScript files used for each interactive plot, copy them into a specified directory, and then set a hook that would search all the chunks until it found the last one and load them all at once at the end.

It turns out that `knitr` has already implemented the tools for this, and that the original `htmlwidgets` package was already doing something relatively similar (I think).  knitr takes the output from each expression in a code block, calls `knit_print()` on each of them, and then decides what to do with the output of each of these when deciding what to actually show in the document.[^2]

knitr's `knit_print()` varies based on the class of the object it's called on, and let's package creators create their own version of `knit_print()` for their own custom objects. For example, `htmlwidgets` has `htmlwidgets:::knit_print.htmlwidget()`, which tells knitr how to print HTML widgets. knitr also lets you bundle additional information (metadata) about the object in the output of `knit_print()` in with the object itself. knitr then collects all this metadata from everything it prints, just like my stateful printing function did, when it collected dependency information from each plotly plot.  

The thing is, `htwmlwidgets` already does this *exact* same thing with `knit_meta_add()`---you can access all the dependencies used in the document with `knit_meta()`.  I realized that I could use `knit_meta('html_dependency')` to get the dependencies and [the end-of-document hook I mentioned earlier](#beep) to add them in at the end.

Compare this:


{% highlight r %}
set_widget_hooks <- function(dep_dir, base_path, 
                             hrefFilter = function(x) paste0("/", x)) {
  
  # Move the dependencies into the specified folder,
  #  makes them relative to the base directory,
  #  and outputs the HTML that loads them.
  render_deps <- function() {
    l <- knitr::knit_meta(class = "html_dependency",
                          clean = FALSE)
    if (length(l) > 0)
      dir.create(dep_dir, showWarnings = FALSE, recursive = TRUE)
    l <- lapply(unique(l), function(dep) {
      dep <- htmltools::copyDependencyToDir(dep, dep_dir, FALSE)
      dep <- htmltools::makeDependencyRelative(dep, base_path, FALSE)
      dep } )
    l <- htmltools::renderDependencies(l,  hrefFilter=hrefFilter)
    htmltools::htmlPreserve(l)
  }
  
  # Adds the dependency-loading HTML at the end of the doc,
  #  without upsetting the previous doc-hook.
  prev_doc_hook <- knitr::knit_hooks$get("document")
  knitr::knit_hooks$set(document = function(x) {
    prev_doc_hook(append(x, render_deps()))
  })
  
  # Sets the default of all chunks to not force
  #  screenshots. You can change it to `TRUE`
  #  on the chunks you want it to screenshot.
  knitr::opts_chunk$set(screenshot.force=FALSE)
}
{% endhighlight %}

to [the original version of my code here]({{ site.baseurl }}{% post_url 2020-04-04-plotly_with_jekyll %}#finalversion).  It's so much better, and applies to all HTML widgets.

You might notice that I set the `screenshot.force` global chunk option to `FALSE`.  Currently, there's no way around doing something like this---knitr checks for HTML widgets to replace with screenshots before it lets custom `knit_print` functions do their thing.[^3]  If you really want to screenshot one widget in particular, you can always set `screenshot.force=TRUE` for that particular chunk.


{% highlight r %}
library(ggplot2)
library(plotly)

ggplot_plot <- iris %>%
  ggplot(aes(Petal.Length, Petal.Width, color=Species)) + 
  geom_point()

plotly_plot <- ggplotly(ggplot_plot)

plotly_plot
{% endhighlight %}

<div class="figure">
<!--html_preserve--><div id="htmlwidget-c31e2e8db18dc616d869" style="width:100%;height:400px;" class="plotly html-widget"></div>
<script type="application/json" data-for="htmlwidget-c31e2e8db18dc616d869">{"x":{"data":[{"x":[1.4,1.4,1.3,1.5,1.4,1.7,1.4,1.5,1.4,1.5,1.5,1.6,1.4,1.1,1.2,1.5,1.3,1.4,1.7,1.5,1.7,1.5,1,1.7,1.9,1.6,1.6,1.5,1.4,1.6,1.6,1.5,1.5,1.4,1.5,1.2,1.3,1.4,1.3,1.5,1.3,1.3,1.3,1.6,1.9,1.4,1.6,1.4,1.5,1.4],"y":[0.2,0.2,0.2,0.2,0.2,0.4,0.3,0.2,0.2,0.1,0.2,0.2,0.1,0.1,0.2,0.4,0.4,0.3,0.3,0.3,0.2,0.4,0.2,0.5,0.2,0.2,0.4,0.2,0.2,0.2,0.2,0.4,0.1,0.2,0.2,0.2,0.2,0.1,0.2,0.2,0.3,0.3,0.2,0.6,0.4,0.3,0.2,0.2,0.2,0.2],"text":["Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.7<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.1<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.2<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.7<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.7<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.0<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.7<br />Petal.Width: 0.5<br />Species: setosa","Petal.Length: 1.9<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.2<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.6<br />Species: setosa","Petal.Length: 1.9<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(248,118,109,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(248,118,109,1)"}},"hoveron":"points","name":"setosa","legendgroup":"setosa","showlegend":true,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[4.7,4.5,4.9,4,4.6,4.5,4.7,3.3,4.6,3.9,3.5,4.2,4,4.7,3.6,4.4,4.5,4.1,4.5,3.9,4.8,4,4.9,4.7,4.3,4.4,4.8,5,4.5,3.5,3.8,3.7,3.9,5.1,4.5,4.5,4.7,4.4,4.1,4,4.4,4.6,4,3.3,4.2,4.2,4.2,4.3,3,4.1],"y":[1.4,1.5,1.5,1.3,1.5,1.3,1.6,1,1.3,1.4,1,1.5,1,1.4,1.3,1.4,1.5,1,1.5,1.1,1.8,1.3,1.5,1.2,1.3,1.4,1.4,1.7,1.5,1,1.1,1,1.2,1.6,1.5,1.6,1.5,1.3,1.3,1.3,1.2,1.4,1.2,1,1.3,1.2,1.3,1.3,1.1,1.3],"text":["Petal.Length: 4.7<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.9<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.6<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.7<br />Petal.Width: 1.6<br />Species: versicolor","Petal.Length: 3.3<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.6<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 3.9<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 3.5<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.2<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.7<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 3.6<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.4<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.1<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 3.9<br />Petal.Width: 1.1<br />Species: versicolor","Petal.Length: 4.8<br />Petal.Width: 1.8<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.9<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.7<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 4.3<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.4<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 4.8<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 5.0<br />Petal.Width: 1.7<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 3.5<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 3.8<br />Petal.Width: 1.1<br />Species: versicolor","Petal.Length: 3.7<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 3.9<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 5.1<br />Petal.Width: 1.6<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.6<br />Species: versicolor","Petal.Length: 4.7<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.4<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.1<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.4<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 4.6<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 3.3<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.2<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.2<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 4.2<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.3<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 3.0<br />Petal.Width: 1.1<br />Species: versicolor","Petal.Length: 4.1<br />Petal.Width: 1.3<br />Species: versicolor"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(0,186,56,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(0,186,56,1)"}},"hoveron":"points","name":"versicolor","legendgroup":"versicolor","showlegend":true,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[6,5.1,5.9,5.6,5.8,6.6,4.5,6.3,5.8,6.1,5.1,5.3,5.5,5,5.1,5.3,5.5,6.7,6.9,5,5.7,4.9,6.7,4.9,5.7,6,4.8,4.9,5.6,5.8,6.1,6.4,5.6,5.1,5.6,6.1,5.6,5.5,4.8,5.4,5.6,5.1,5.1,5.9,5.7,5.2,5,5.2,5.4,5.1],"y":[2.5,1.9,2.1,1.8,2.2,2.1,1.7,1.8,1.8,2.5,2,1.9,2.1,2,2.4,2.3,1.8,2.2,2.3,1.5,2.3,2,2,1.8,2.1,1.8,1.8,1.8,2.1,1.6,1.9,2,2.2,1.5,1.4,2.3,2.4,1.8,1.8,2.1,2.4,2.3,1.9,2.3,2.5,2.3,1.9,2,2.3,1.8],"text":["Petal.Length: 6.0<br />Petal.Width: 2.5<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 5.9<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.8<br />Petal.Width: 2.2<br />Species: virginica","Petal.Length: 6.6<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 4.5<br />Petal.Width: 1.7<br />Species: virginica","Petal.Length: 6.3<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.8<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 6.1<br />Petal.Width: 2.5<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 5.3<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 5.5<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 5.0<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 2.4<br />Species: virginica","Petal.Length: 5.3<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.5<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 6.7<br />Petal.Width: 2.2<br />Species: virginica","Petal.Length: 6.9<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.0<br />Petal.Width: 1.5<br />Species: virginica","Petal.Length: 5.7<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 4.9<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 6.7<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 4.9<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.7<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 6.0<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 4.8<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 4.9<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 5.8<br />Petal.Width: 1.6<br />Species: virginica","Petal.Length: 6.1<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 6.4<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 2.2<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 1.5<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 1.4<br />Species: virginica","Petal.Length: 6.1<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 2.4<br />Species: virginica","Petal.Length: 5.5<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 4.8<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.4<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 2.4<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 5.9<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.7<br />Petal.Width: 2.5<br />Species: virginica","Petal.Length: 5.2<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.0<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 5.2<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 5.4<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 1.8<br />Species: virginica"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(97,156,255,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(97,156,255,1)"}},"hoveron":"points","name":"virginica","legendgroup":"virginica","showlegend":true,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null}],"layout":{"margin":{"t":24.8556800885568,"r":6.6417600664176,"b":25.4600802546008,"l":23.2461602324616},"plot_bgcolor":"rgba(255,255,255,1)","paper_bgcolor":"rgba(255,255,255,1)","font":{"color":"rgba(0,0,0,1)","family":"","size":13.2835201328352},"xaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[0.705,7.195],"tickmode":"array","ticktext":[null],"categoryorder":"array","categoryarray":[null],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.3208800332088,"tickwidth":0.603796369674327,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":10.6268161062682},"tickangle":-0,"showline":true,"linecolor":"rgba(0,0,0,1)","linewidth":0.603796369674327,"showgrid":false,"gridcolor":null,"gridwidth":0,"zeroline":false,"anchor":"y","title":{"text":"Petal.Length","font":{"color":"rgba(0,0,0,1)","family":"","size":13.2835201328352}},"hoverformat":".2f"},"yaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-0.02,2.62],"tickmode":"array","ticktext":[null],"categoryorder":"array","categoryarray":[null],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.3208800332088,"tickwidth":0.603796369674327,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":10.6268161062682},"tickangle":-0,"showline":true,"linecolor":"rgba(0,0,0,1)","linewidth":0.603796369674327,"showgrid":false,"gridcolor":null,"gridwidth":0,"zeroline":false,"anchor":"x","title":{"text":"Petal.Width","font":{"color":"rgba(0,0,0,1)","family":"","size":13.2835201328352}},"hoverformat":".2f"},"shapes":[{"type":"rect","fillcolor":null,"line":{"color":null,"width":0,"linetype":[]},"yref":"paper","xref":"paper","x0":0,"x1":1,"y0":0,"y1":1}],"showlegend":true,"legend":{"bgcolor":"rgba(255,255,255,1)","bordercolor":"transparent","borderwidth":1.71796707229778,"font":{"color":"rgba(0,0,0,1)","family":"","size":10.6268161062682},"y":0.934383202099738},"annotations":[{"text":"Species","x":1.02,"y":1,"showarrow":false,"ax":0,"ay":0,"font":{"color":"rgba(0,0,0,1)","family":"","size":13.2835201328352},"xref":"paper","yref":"paper","textangle":-0,"xanchor":"left","yanchor":"bottom","legendTitle":true}],"hovermode":"closest","barmode":"relative"},"config":{"doubleClick":"reset","showSendToCloud":false},"source":"A","attrs":{"b2e013b28bad":{"x":{},"y":{},"colour":{},"type":"scatter"}},"cur_data":"b2e013b28bad","visdat":{"b2e013b28bad":["function (y) ","x"]},"highlight":{"on":"plotly_click","persistent":false,"dynamic":false,"selectize":false,"opacityDim":0.2,"selected":{"opacity":1},"debounce":0},"shinyEvents":["plotly_hover","plotly_click","plotly_selected","plotly_relayout","plotly_brushed","plotly_brushing","plotly_clickannotation","plotly_doubleclick","plotly_deselect","plotly_afterplot"],"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script><!--/html_preserve-->
<p class="caption">My knitr hooks in action!</p>
</div>


<hr />
<br />

## Source Code:

> [`knit_hook_setter.R`](https://gist.github.com/burchill/8392b2a753652e24a35a8a1dd707c1b1)

This is the better, _improved_ version of the HTML widget enabler.

### Footnotes:

[^1]: Previously, I had attempted to play a sound _after_ the last chunk of the document, by setting a global chunk option and hook that would check the label of each chunk to see if it was the same as the last chunk's, and play the sound after that. Unfortunately, cached chunks are never checked this way, so if the last chunk was cached, the sound would never play.

[^2]: As far as I can gather! This is definitely an oversimplication at best though.

[^3]: You _could_ change the global `render` chunk option to a function that automatically checks if an object is a widget, and then temporarily disables screenshotting before passing it to `knit_print()`, but that's less straightforward and unnecessary complexity.
<!--html_preserve--><script src="/_posts/figures/generated/html_dependencies/htmlwidgets-1.3/htmlwidgets.js"></script>
<script src="/_posts/figures/generated/html_dependencies/plotly-binding-4.9.0/plotly.js"></script>
<script src="/_posts/figures/generated/html_dependencies/typedarray-0.1/typedarray.min.js"></script>
<script src="/_posts/figures/generated/html_dependencies/jquery-1.11.3/jquery.min.js"></script>
<link href="/_posts/figures/generated/html_dependencies/crosstalk-1.0.0/css/crosstalk.css" rel="stylesheet" />
<script src="/_posts/figures/generated/html_dependencies/crosstalk-1.0.0/js/crosstalk.min.js"></script>
<link href="/_posts/figures/generated/html_dependencies/plotly-htmlwidgets-css-1.46.1/plotly-htmlwidgets.css" rel="stylesheet" />
<script src="/_posts/figures/generated/html_dependencies/plotly-main-1.46.1/plotly-latest.min.js"></script><!--/html_preserve-->
<!--html_preserve--><!--zoption version:0.01--><!--/html_preserve-->
