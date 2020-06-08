---
layout: post
title: "How to Use Plotly/HTML Widgets in Jekyll the RIGHT Way"
comments:  true
published:  true
author: "Zachary Burchill"
date: 2020-04-04 00:30:00
permalink: /plotly_with_jekyll/
categories: [R,knitr,plotly,"interactive plots","data visualization","Jekyll","GitHub Pages"]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---








[Plotly](https://plotly.com/), which lets you interact with data and plots in incredibly pleasing ways (see [this post by my brother and I for examples]({{ site.baseurl }}{% post_url 2020-03-27-questionable_movies %})) offers [a _load_ of cool possibilities with R](https://plotly.com/r/), whether you want dashboards or engaging data visualizations.  It's super web-friendly and fits like a glove into workflows that knit HTML.

The only problem is that you're basically screwed if you want to use Plotly (or any HTML widgets) with [Jekyll or GitHub Pages](https://jekyllrb.com/docs/github-pages/).  Sure, there are ways you can do it, but they're enormously hacky and would lead to an *insane* posting workflow. In this post, I will show you how to do it the *right* way.

<!--more-->

## Everbody else is wrong

**_\~\~UPDATE: <u>Including</u> me, evidently! [See the very end of this post!](#update)\~\~_**

Yeah, you heard it.  There are numerous blog posts and tidbits out there about using Plotly and HTML widgets with Jekyll, and you should resent every single one of them for being hacky as hell.  Let's go through a few:

 * [There's this post by Saul Cruz](https://saulcruzr.github.io/Plotly_Example/) wants you to do a two-step process where you first knit a file from R Markdown to HTML (individually) and then have another Markdown file load _that_ file.
 * [There's a post by Ryan Kuhn](http://ryankuhn.net/blog/How-To-Use-Plotly-With-Jekyll) that basically suggests writing everything in HTML rather than Markdown, essentially defeating the point of even _using_ Markdown.
 * [There's this relatively advanced post by Gervasio Marchand](https://g3rv4.com/2017/08/htmlwidgets-jekyll-rstats) that advocates doing something a little like what Saul suggested, but in a much friendlier, well-thought-out way.  Still needleslly complicated though.
 
So yeah, but nah.  I'm here to give you the easy, super-sexy way.

## So what's the problem again?

Oh yeah. Let's get to that.  First, let's make an example ggplot, which works fine in R Markdown -> Jekyll.


{% highlight r %}
library(ggplot2)
library(plotly)

# Make a super simple plot
p <- iris %>%
  ggplot(aes(Petal.Length, Petal.Width, color=Species)) + 
  geom_point()
p
{% endhighlight %}

<img src="/_posts/figures/generated/source/x2020-04-04-plotly_with_jekyll/unnamed-chunk-2-1.png" title="This is a normal ggplot plot, booooring" alt="This is a normal ggplot plot, booooring" width="80%" />

<p class='figcaption'>This is a normal ggplot plot, booooring</p>

Now let's use the `ggplotly()` function from the `plotly` package to convert the ggplot into a plotly plot:


{% highlight r %}
# Convert it into a plotly plot
p <- ggplotly(p)
p
{% endhighlight %}



{% highlight text %}
## PhantomJS not found. You can install it with webshot::install_phantomjs(). If it is installed, please make sure the phantomjs executable can be found via the PATH variable.
{% endhighlight %}



{% highlight text %}
## Warning in normalizePath(f2): path[1]="webshotafdc31bcc3ea.png": No such file or
## directory
{% endhighlight %}



{% highlight text %}
## Warning in file(con, "rb"): cannot open file 'webshotafdc31bcc3ea.png': No such
## file or directory
{% endhighlight %}



{% highlight text %}
## Error in file(con, "rb"): cannot open the connection
{% endhighlight %}

_**Oops!**_  

It turns out that when `knitr` sees that you're trying to use an HTML widget in a non-HTML output, it actually tries to open it with a web browser, take a screenshot of it with [`webshot`](https://github.com/wch/webshot), and then use that. I don't have a necessary component of that package installed, so it throws an error. Even if it had used a picture, that's not what we want it to do!

## The basic solution

After digging around in the source code from a few packages (what ended up helping the most was the `saveWidget()` function from the `htmlwidgets` package), I finally got a grip on what was up.  A plotly plot has two major components to it: the HTML that instantiates it, and the Javascript that makes it run.  

## The HTML

Getting the HTML wasn't that hard, you can do something like the following in a normal R chunk:


{% highlight r %}
render_plotly_html <- function(p) {
  p %>%
    plotly::as_widget() %>%
    list() %>%
    htmltools::tagList()
}
{% endhighlight %}

Unfortunately, you'll just end up with an empty place where the plot _should_ be.  You still need the Javascript.  And that's definitely the more annoying part.

## The Javascript

Normally, the Javascript used to power HTML widgets and plotly plots is already saved in these packages on your computer. When you view the plots from, say, RStudio, it just adds HTML elements that load the scripts in from where they are on your computer, something like `<script scr="path-to-script"></script>`. 

If you want to save a widget and share it with a friend (who doesn't have the same Javascript files as you) `htmlwidgets::saveWidget()` will let you essentially smush all the disparate Javascript files so that they're hardcoded _into_ the HTML file, along with the data, and saves that.

### A (bad) first step

And my first attempt at solving this problem was to make code that would basically do just that---automatically save each plotly widget as a standalone HTML file, and load it in through an `<iframe>` element.  But that's definitely not the ideal situation: you have to redundantly save Javascript dependencies (and load them), and the iframe looks ugly and makes you have to do scrolling stuff.  

After _really_ unspooling the `saveWidget()` source code, I had a better understanding of how dependencies were being handled, and I noticed that when you didn't smush all the Javascript files into a standalone HTML file, it would "uproot" all the dependencies, copy them to a specified folder, and add them in to the HTML as links. I made my own version:


{% highlight r %}
get_deps <- function(
  widget, # The widget in question
  postdir, # The path to the posts' content data
  basedir, # The base directory of my GH Pages Jekyll repo
  libdirname = "js_files/" # A subdirectory for the JS files
  ) {
  libdir <- paste0(postdir, libdirname)
  dir.create(libdir, showWarnings = FALSE, recursive = TRUE)
  
  # This gets the dependencies from the widget
  deps <- htmltools::renderTags(widget)$dependencies %>%
    # For every dependency...
    lapply(function(dep) {
      # Copy it to the post's directory
      htmltools::copyDependencyToDir(dep, libdir, FALSE) %>%
        # Adjust it so that the path is relative
        htmltools::makeDependencyRelative(basedir, FALSE)
    })
}

# Turns the dependencies into HTML
render_deps <- function(deps) {
  deps %>%
    # Turns the deps into HTML
    htmltools::renderDependencies(
      # See explanation in text below
      hrefFilter = function(x) paste0("/",x)) %>%
    # Helps preserve the HTML just in case
    htmltools::htmlPreserve()
}
{% endhighlight %}

Let me explain that "postdir" and "basedir" stuff, the "postdir" is the directory that corresponds to the posts' `_posts/` subdirectory, or wherever you want to keep its automatically generated content, like plot images.  The "basedir" variable needs to be supplied because you need to know where the actual post itself is going to be in order to make the links right.  What these variables are will totally depend on your setup and how you organize your files, but should be easy to tweak.

I was able to add them as default knitr variables [by adding them into my `build.R` file as `plotly.savepath` and `proj.basedir` via `knitr::opts_chunk$set()`.](https://github.com/burchill/burchill.github.io/commit/ce18ff7ee833d4fc745cdd529f9e5035fb3a442d#diff-7d179ec4956ea309f110b6105874d871)

Notice, however, the `hrefFilter` function in `renderDependencies`.  I noticed that the output of my dependencies, after I made them relative, started like, `<script src="_posts/...`, which didn't actually work. I needed to add an extra slash in front of the relative path for it to work (i.e., `<script src="/_posts/...`). The `hrefFilter` argument is a function that puts that finishing touch on.

Anyway, I could now generate the correct HTML links for the dependencies for each plotly plot, doing something like:


{% highlight r %}
HTML <- p %>%
  get_deps(
    postdir="~/burchill.github.io/_posts/figures/generated/source/x2020-04-04-plotly_with_jekyll/",
    basedir="~/burchill.github.io/") %>% 
  render_deps()
{% endhighlight %}

In order to get `knitr` to render the HTML properly though, I had to make the chunk knew to not mess with the output, setting the `results` parameter to `"asis"`.


{% highlight text %}
```{r, results="asis"}
cat(HTML)
render_plotly_html(p)
```
{% endhighlight %}

Unfortunately, this meant either redundantly adding `<script>` HTML elements every time you wanted to display a widget, or hoping that every widget has the same dependencies.[^1]  A "real" right way would only save/load the minimal amount of Javascript files the minimal number of times.

But that would mean collecting all the dependencies, and only rendering them at the end.  Can we do that?

Yes.

## Function factories and R environments

There are a number of ways you could imagine counting and accumulating all the Javascript dependencies: you could use global variables, you could push the data into `knitr` variables, etc.  I first thought about just using global variables, but I knew that would become messy and error-prone, especially if I had to continue the practice across many different posts.

I'm not going to get into _all_ the nitty-gritty details here, but I decided to use something called a "function factory", that is, a function that returns other functions.  The way R works is that each function call makes its own mini-environment, both when it is called and when it is defined.  Look at the `inner_fn` in the code below: it is defined such that the `counter` variable it uses comes from the environment above it---one that is created when `function_factory()` is called.


{% highlight r %}
function_factory <- function() {
  counter <- 0
  inner_fn <- function() {
    print(counter)
    # The `<<-` does assignment for variables in higher environments
    counter <<- counter + 1
  }
  return(inner_fn)
}
fn <- function_factory()
{% endhighlight %}

The environment that the `inner_fn` is created in essentially "travels with" the function, and the `<<-` operator lets `inner_fn` change variables in that environment. It has become a "stateful" function, in that it has a state associated with it (the state that holds `counter`).  See how it keeps track of `counter` each time it is called:


{% highlight r %}
fn()
{% endhighlight %}



{% highlight text %}
## [1] 0
{% endhighlight %}



{% highlight r %}
fn()
{% endhighlight %}



{% highlight text %}
## [1] 1
{% endhighlight %}



{% highlight r %}
fn()
{% endhighlight %}



{% highlight text %}
## [1] 2
{% endhighlight %}

I figured I could create a stateful function for displaying HTML widgets, that keeps track of all the dependencies of the widgets it displays, accumulating them as it displays them.

Something like:


{% highlight r %}
plotly_collector_maker <- function() {
  deps <- list()
  function(p=NULL) {
    # If you don't give it a plot to take dependencies from,
    #   it returns the unique set
    if (!is.null(p)) {
      deps <<- append(deps, htmltools::renderTags(p)$dependencies)
      invisible(NULL) 
    } else {
      unique(deps)
    }
  }
}
plotly_collector <- plotly_collector_maker()
plotly_collector(p)
{% endhighlight %}

I could go around using `plotly_collector()` to get all the dependencies, and I could then add a code block at the end that would turn them into the right HTML and have that load the Javascript.

But I could do even better than that. I wanted to make it so that it would _automatically_ load the JS dependencies for me.

## Automating the final JS loading

My first move was to see if I could programmatically create a chunk at the end of the document, and put the code in there. `knitr` is _incredibly_ powerful, so that's not out of the question.  Unfortunately, I didn't find a way to do that without some very hacky workarounds. But After immersing myself in `knitr` long enough, I realized I could access the last chunk in the document by using `knitr::all_labels()`, which would return me the labels of each chunk, in order of appearance.

Then, I could make a `knitr` hook would check every chunk if its label matched the label of the last chunk.  I could then have it spit out the HTML, after it evaluated the last chunk.


{% highlight r %}
# Get the last label
# My cringey `._` naming is because I want to avoid
#   common global variable names
._plotly_last_label <- tail(knitr::all_labels(), n=1)[[1]]

# Make a hook that, if it's after the last chunk,
# Spits out the dependencies
knitr::knit_hooks$set(._plotly_checker = 
                        function(before, options) {
  if (options$label == ._plotly_last_label & !before)
    # Remember, plotly_collector() returns 
    #   the collected dependencies
    render_deps(plotly_collector())
})
# Sets the options for every chunk so the hook will be run on them  
knitr::opts_chunk$set(._plotly_checker = TRUE)
{% endhighlight %}

The cool thing about returning strings before and after code chunks (i.e., the output of the `._plotly_checker` function) is that you don't need to have the `results="asis"`---they're automatically treated "as-is", regardless of how the output for that chunk is treated.

But even this is still not clean enough.  Even though I named the global variables names that no one in their right mine could accidentally write over, they're still a bunch of gloval variables lying all gross everywhere, eww so gross.

In order to make things "cleaner", I decided I could make a "multi-function factory" that would create objects that had multiple stateful functions that all referred to the same state.[^2]  My idea was that I could use the same object to give me both an automated hook function _and_ the plotting function. This is what it would be conceptually:


{% highlight r %}
plotly_obj_maker <- function() {
  deps <- list()
  hook_fn <- function(before, options) {...}
  plot_fn <- function(p) {...}
  # I didn't really use the get/set fns, they just show
  #   how analogous this system is to a Python class
  set_deps <- function(newdeps) deps <<- newdeps
  get_deps <- function() return(deps)
  
  list(
    hook=hook_fn, plot=plot_fn,
    set_deps=set_deps, get_deps=get_deps
  )
}
plotly_obj <- plotly_obj_maker()

# You can set the hook...
knitr::knit_hooks$set(._plotly_checker = plotly_obj$hook)
# ...and plot with a single function 
plotly_obj$plot(p)
{% endhighlight %}

## Putting it all together {#finalversion}

I eventually decided that the only function I *really* needed to surface was the plotting function---everything else could be taken care of behind the scenes, without really reducing important use cases.  I boiled it down to the following:


{% highlight r %}
plotly_manager <- function(
  postdir = knitr::opts_chunk$get("plotly.savepath"), 
  basedir = knitr::opts_chunk$get("proj.basedir"),
  libdirname = "js_files/",
  hrefFilter = function(x) paste0("/", x)) {
  
  last_label <- tail(knitr::all_labels(), n=1)[[1]]
  deps <- list()
  libdir <- paste0(postdir, libdirname)
  
  render_deps <- function(l) {
    if (length(l) > 0)
      dir.create(libdir, showWarnings = FALSE, recursive = TRUE)
    l <- lapply(unique(l), function(dep) {
      dep <- htmltools::copyDependencyToDir(dep, libdir, FALSE)
      dep <- htmltools::makeDependencyRelative(dep, basedir, FALSE)
      dep } )
    l <- htmltools::renderDependencies(l, hrefFilter=hrefFilter)
    htmltools::htmlPreserve(l)
  }
  
  add_deps_from_plot <- function(p) {
    deps <<- append(deps, htmltools::renderTags(p)$dependencies)
  }
  
  hook <- function(before, options) {
    if (options$label == last_label & !before)
      render_deps(deps)
  }
  
  plot_plotly <- function(p) {
    add_deps_from_plot(p)
    htmltools::tagList(list(plotly::as_widget(p)))
  }
  
  knitr::knit_hooks$set(._plotly_checker = hook)
  knitr::opts_chunk$set(._plotly_checker = TRUE)
  
  plot_plotly
}
{% endhighlight %}

If I include this single function in a source file or in an early chunk, all I have to do is the following to get a plotting function that will _automatically_ collect all the dependencies, _automatically_ save the right dependencies to the post's generated source directory, and _automatically_ add the minimal amount of dependencies at the end of the last chunk.  All you have to do is:


{% highlight r %}
plot_plotly <- plotly_manager()
{% endhighlight %}

And then you can use `plot_plotly()` anywhere to use any plotly plot you want, whenever:


{% highlight r %}
plot_plotly(p)
{% endhighlight %}

<!--html_preserve--><div id="htmlwidget-ecd0432b4be5d8821cf5" style="width:100%;height:400px;" class="plotly html-widget"></div>
<script type="application/json" data-for="htmlwidget-ecd0432b4be5d8821cf5">{"x":{"data":[{"x":[1.4,1.4,1.3,1.5,1.4,1.7,1.4,1.5,1.4,1.5,1.5,1.6,1.4,1.1,1.2,1.5,1.3,1.4,1.7,1.5,1.7,1.5,1,1.7,1.9,1.6,1.6,1.5,1.4,1.6,1.6,1.5,1.5,1.4,1.5,1.2,1.3,1.4,1.3,1.5,1.3,1.3,1.3,1.6,1.9,1.4,1.6,1.4,1.5,1.4],"y":[0.2,0.2,0.2,0.2,0.2,0.4,0.3,0.2,0.2,0.1,0.2,0.2,0.1,0.1,0.2,0.4,0.4,0.3,0.3,0.3,0.2,0.4,0.2,0.5,0.2,0.2,0.4,0.2,0.2,0.2,0.2,0.4,0.1,0.2,0.2,0.2,0.2,0.1,0.2,0.2,0.3,0.3,0.2,0.6,0.4,0.3,0.2,0.2,0.2,0.2],"text":["Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.7<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.1<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.2<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.7<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.7<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.0<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.7<br />Petal.Width: 0.5<br />Species: setosa","Petal.Length: 1.9<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.2<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.1<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.3<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.6<br />Species: setosa","Petal.Length: 1.9<br />Petal.Width: 0.4<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.3<br />Species: setosa","Petal.Length: 1.6<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.5<br />Petal.Width: 0.2<br />Species: setosa","Petal.Length: 1.4<br />Petal.Width: 0.2<br />Species: setosa"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(248,118,109,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(248,118,109,1)"}},"hoveron":"points","name":"setosa","legendgroup":"setosa","showlegend":true,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[4.7,4.5,4.9,4,4.6,4.5,4.7,3.3,4.6,3.9,3.5,4.2,4,4.7,3.6,4.4,4.5,4.1,4.5,3.9,4.8,4,4.9,4.7,4.3,4.4,4.8,5,4.5,3.5,3.8,3.7,3.9,5.1,4.5,4.5,4.7,4.4,4.1,4,4.4,4.6,4,3.3,4.2,4.2,4.2,4.3,3,4.1],"y":[1.4,1.5,1.5,1.3,1.5,1.3,1.6,1,1.3,1.4,1,1.5,1,1.4,1.3,1.4,1.5,1,1.5,1.1,1.8,1.3,1.5,1.2,1.3,1.4,1.4,1.7,1.5,1,1.1,1,1.2,1.6,1.5,1.6,1.5,1.3,1.3,1.3,1.2,1.4,1.2,1,1.3,1.2,1.3,1.3,1.1,1.3],"text":["Petal.Length: 4.7<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.9<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.6<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.7<br />Petal.Width: 1.6<br />Species: versicolor","Petal.Length: 3.3<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.6<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 3.9<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 3.5<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.2<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.7<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 3.6<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.4<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.1<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 3.9<br />Petal.Width: 1.1<br />Species: versicolor","Petal.Length: 4.8<br />Petal.Width: 1.8<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.9<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.7<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 4.3<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.4<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 4.8<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 5.0<br />Petal.Width: 1.7<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 3.5<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 3.8<br />Petal.Width: 1.1<br />Species: versicolor","Petal.Length: 3.7<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 3.9<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 5.1<br />Petal.Width: 1.6<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.5<br />Petal.Width: 1.6<br />Species: versicolor","Petal.Length: 4.7<br />Petal.Width: 1.5<br />Species: versicolor","Petal.Length: 4.4<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.1<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.4<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 4.6<br />Petal.Width: 1.4<br />Species: versicolor","Petal.Length: 4.0<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 3.3<br />Petal.Width: 1.0<br />Species: versicolor","Petal.Length: 4.2<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.2<br />Petal.Width: 1.2<br />Species: versicolor","Petal.Length: 4.2<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 4.3<br />Petal.Width: 1.3<br />Species: versicolor","Petal.Length: 3.0<br />Petal.Width: 1.1<br />Species: versicolor","Petal.Length: 4.1<br />Petal.Width: 1.3<br />Species: versicolor"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(0,186,56,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(0,186,56,1)"}},"hoveron":"points","name":"versicolor","legendgroup":"versicolor","showlegend":true,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[6,5.1,5.9,5.6,5.8,6.6,4.5,6.3,5.8,6.1,5.1,5.3,5.5,5,5.1,5.3,5.5,6.7,6.9,5,5.7,4.9,6.7,4.9,5.7,6,4.8,4.9,5.6,5.8,6.1,6.4,5.6,5.1,5.6,6.1,5.6,5.5,4.8,5.4,5.6,5.1,5.1,5.9,5.7,5.2,5,5.2,5.4,5.1],"y":[2.5,1.9,2.1,1.8,2.2,2.1,1.7,1.8,1.8,2.5,2,1.9,2.1,2,2.4,2.3,1.8,2.2,2.3,1.5,2.3,2,2,1.8,2.1,1.8,1.8,1.8,2.1,1.6,1.9,2,2.2,1.5,1.4,2.3,2.4,1.8,1.8,2.1,2.4,2.3,1.9,2.3,2.5,2.3,1.9,2,2.3,1.8],"text":["Petal.Length: 6.0<br />Petal.Width: 2.5<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 5.9<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.8<br />Petal.Width: 2.2<br />Species: virginica","Petal.Length: 6.6<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 4.5<br />Petal.Width: 1.7<br />Species: virginica","Petal.Length: 6.3<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.8<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 6.1<br />Petal.Width: 2.5<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 5.3<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 5.5<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 5.0<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 2.4<br />Species: virginica","Petal.Length: 5.3<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.5<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 6.7<br />Petal.Width: 2.2<br />Species: virginica","Petal.Length: 6.9<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.0<br />Petal.Width: 1.5<br />Species: virginica","Petal.Length: 5.7<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 4.9<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 6.7<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 4.9<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.7<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 6.0<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 4.8<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 4.9<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 5.8<br />Petal.Width: 1.6<br />Species: virginica","Petal.Length: 6.1<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 6.4<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 2.2<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 1.5<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 1.4<br />Species: virginica","Petal.Length: 6.1<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 2.4<br />Species: virginica","Petal.Length: 5.5<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 4.8<br />Petal.Width: 1.8<br />Species: virginica","Petal.Length: 5.4<br />Petal.Width: 2.1<br />Species: virginica","Petal.Length: 5.6<br />Petal.Width: 2.4<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 5.9<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.7<br />Petal.Width: 2.5<br />Species: virginica","Petal.Length: 5.2<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.0<br />Petal.Width: 1.9<br />Species: virginica","Petal.Length: 5.2<br />Petal.Width: 2.0<br />Species: virginica","Petal.Length: 5.4<br />Petal.Width: 2.3<br />Species: virginica","Petal.Length: 5.1<br />Petal.Width: 1.8<br />Species: virginica"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(97,156,255,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(97,156,255,1)"}},"hoveron":"points","name":"virginica","legendgroup":"virginica","showlegend":true,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null}],"layout":{"margin":{"t":25.7412480974125,"r":7.30593607305936,"b":28.0060882800609,"l":25.5707762557078},"plot_bgcolor":"rgba(255,255,255,1)","paper_bgcolor":"rgba(255,255,255,1)","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187},"xaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[0.705,7.195],"tickmode":"array","ticktext":[null],"categoryorder":"array","categoryarray":[null],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":true,"linecolor":"rgba(0,0,0,1)","linewidth":0.66417600664176,"showgrid":false,"gridcolor":null,"gridwidth":0,"zeroline":false,"anchor":"y","title":{"text":"Petal.Length","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"yaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-0.02,2.62],"tickmode":"array","ticktext":[null],"categoryorder":"array","categoryarray":[null],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":true,"linecolor":"rgba(0,0,0,1)","linewidth":0.66417600664176,"showgrid":false,"gridcolor":null,"gridwidth":0,"zeroline":false,"anchor":"x","title":{"text":"Petal.Width","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"shapes":[{"type":"rect","fillcolor":null,"line":{"color":null,"width":0,"linetype":[]},"yref":"paper","xref":"paper","x0":0,"x1":1,"y0":0,"y1":1}],"showlegend":true,"legend":{"bgcolor":"rgba(255,255,255,1)","bordercolor":"transparent","borderwidth":1.88976377952756,"font":{"color":"rgba(0,0,0,1)","family":"","size":11.689497716895},"y":0.927821522309711},"annotations":[{"text":"Species","x":1.02,"y":1,"showarrow":false,"ax":0,"ay":0,"font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187},"xref":"paper","yref":"paper","textangle":-0,"xanchor":"left","yanchor":"bottom","legendTitle":true}],"hovermode":"closest","barmode":"relative"},"config":{"doubleClick":"reset","showSendToCloud":false},"source":"A","attrs":{"afdc27ebead0":{"x":{},"y":{},"colour":{},"type":"scatter"}},"cur_data":"afdc27ebead0","visdat":{"afdc27ebead0":["function (y) ","x"]},"highlight":{"on":"plotly_click","persistent":false,"dynamic":false,"selectize":false,"opacityDim":0.2,"selected":{"opacity":1},"debounce":0},"shinyEvents":["plotly_hover","plotly_click","plotly_selected","plotly_relayout","plotly_brushed","plotly_brushing","plotly_clickannotation","plotly_doubleclick","plotly_deselect","plotly_afterplot"],"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script><!--/html_preserve-->

Essentially _flawless_.

### Addendum

I actually wanted to go even further than this. Normally, as far as I knew, when you just return a object visibly in R, it automatically prints it. For example, when you save a plot to `p` and enter `p` in the console by itself, it prints out the object.  

You can actually change how something is printed out in R by making a `print.<class>` function---for example, `ggplot2` uses the `ggplot2:::print.ggplot()` function so that when you return a ggplot, it displays the plot.

In a simpler world, I could have just replaced the `"plotly"` class print function,


{% highlight r %}
print.plotly <- plot_plotly
{% endhighlight %}

and you wouldn't have to even remember to call `plot_plotly()` to use plotly plots.  And, if you do the above and call `print(p)`, it works!  The only issue is, if you just do:


{% highlight r %}
p
{% endhighlight %}

<!--html_preserve--><script src="/_posts/figures/generated/html_dependencies/htmlwidgets-1.3/htmlwidgets.js"></script>
<script src="/_posts/figures/generated/html_dependencies/plotly-binding-4.9.0/plotly.js"></script>
<script src="/_posts/figures/generated/html_dependencies/typedarray-0.1/typedarray.min.js"></script>
<script src="/_posts/figures/generated/html_dependencies/jquery-1.11.3/jquery.min.js"></script>
<link href="/_posts/figures/generated/html_dependencies/crosstalk-1.0.0/css/crosstalk.css" rel="stylesheet" />
<script src="/_posts/figures/generated/html_dependencies/crosstalk-1.0.0/js/crosstalk.min.js"></script>
<link href="/_posts/figures/generated/html_dependencies/plotly-htmlwidgets-css-1.46.1/plotly-htmlwidgets.css" rel="stylesheet" />
<script src="/_posts/figures/generated/html_dependencies/plotly-main-1.46.1/plotly-latest.min.js"></script><!--/html_preserve-->

`knitr` defaults to its bad `webshot` behavior, evidently bypassing the `print()` function somehow.  If you know how to get around this, please contact me on Twitter or drop me a comment below!

## UPDATE! (2020-06-07) {#update}

To get straight to the point, I finally learned how to do what I described in the addendum above, and also realized all my code is reinventing the wheel. For a cleaner, better, and more correct version of this code, [check out my new post here]({{ site.baseurl }}{% post_url 2020-06-07-knitr_tricks %}#rightplotly)!


<hr />
<br />

## Source Code:

> [`plotly_plot_maker.R`](https://gist.github.com/burchill/9df0f6245ea7768e5b6bbd0a1c22db08)

This is the final version of the code I made for _this_ post.

> [`knit_hook_setter.R`](https://gist.github.com/burchill/8392b2a753652e24a35a8a1dd707c1b1)

This is the better, _improved_ version of the final code I discuss in [my new post]({{ site.baseurl }}{% post_url 2020-06-07-knitr_tricks %}#rightplotly).

### Footnotes:

[^1]: _Technically_, the code I have here probably won't work out of the box with other widgets, since the way I get the plotly HTML is specific to plotly. But it would be trivial to add something that would work with other HTML widgets, and if I ever use them, I'll change that bit.

[^2]: Notice that this basically is an object-oriented class.
