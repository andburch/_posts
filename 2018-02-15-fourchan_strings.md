---
layout:  post
title: "Python Lessons from 4chan, Part 2: Use (the New) String Formatting!"
comments:  true
published:  true
author: "Zach Burchill"
date: 2018-02-15 10:00:00
permalink: /fourchan_strings/
categories: ["python lessons from 4chan",python,'web scraping','4chan','manga','webcomics',"string formatting"]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---



If I wanted to make this post sound professional and industrious, I would say that my motivations behind this project were because I've started working towards my Bayesian model of webcomic updates again, and that I'm taking an intermediate step by analyzing data from similar content creators.

But the truth is, I was just pissed off that I couldn't read the manga I wanted to.

These are the Python lessons I learned scraping manga scanlations off of 4chan.

### Part 2: Using (the _new_) string formatting!

<!--more-->

_For the background on the project this post stems from, as well as some ruminations on logging in Python, check out the [first post in the series]({{ site.baseurl }}{% post_url 2018-02-10-fourchan_logging %})._

## In with the new, out with the old (string formatting method)

Python newbs might be a little confused about something in the logging configuration code from the [previous post in this series]({{ site.baseurl }}{% post_url 2018-02-10-fourchan_logging %}).  Specifically, the line:

```python
format = '%(asctime)s -- %(threadName)s %(levelname)s %(message)s'
```

The weird "%(\<name\>)s" bits above are part of how Python formats strings (i.e., puts variables into string formats). One of my bad habits in Python has been to format strings almost exclusively with '+'s, like: `print("Count is: "+c+"!")`.

This sucks because it's ugly, it raises an error if `c` is not a string, and even if "fixed" by making it `+str(c)+`, it starts getting really long/hard to read.  

But with Python's string formatting, you can make it much *prettier*, more *legible*, and *repeatable*. Python can substitute the values of variables into the string wherever the "%"s occur. But all that is Python's **_old_** way of formatting strings.  The new way is definitely better.

The docs for string formatting [_suck_](https://docs.python.org/3/library/string.html#string-formatting), but some saintly souls have taken it upon themselves to make [a much more user-friendly intro to the topic](https://pyformat.info/), which I **absolutely** recommend. Instead of "%", the new method uses "{}" and `.format()`. 

### Safer, clearer code

Naming the variables in the format string makes understanding the purpose of strings much easier, especially if you are reusing it. By having to specify which formatting names correspond to which variables, you also make the formatting more explicit, less prone to error, and robust to ordering changes.

```python
html_formatter = "<html><head>{head_html}</head><body>{body_html}</body></html>"
page_1 = html_formatter.format(head_html="<title>Page 1!</title>",
                               body_html="<p>This is the first</p>")
print(page_1)
```

```
<html><head><title>Page 1!</title></head><body><p>This is the first</p></body></html>
```

Note that the keyword argument-style of formatting is only available for the new string formatting.

### More flexible, shorter code

Another great advantage to string formatting above simple concatenation is that actually means your code can be much shorter. Instead of casting each variables with `str()`, you can just use `{!s}`.  And if you are bundling information together with dictionaries, it's super easy to get string formatters that can play nice across different dictionaries.

```python
# String formatting plays well with dictionaries
counter_dict = {"title": "CounterDict", "counter": 5, "score": (6.4, 3)}

# Using dictionaries can be easily done two ways:
debug_string1 = "'{title}' has {counter!s} counts and a score of {score[0]!s}"
debug_string2 = "'{d[title]}' has {d[counter]!s} counts and a score of {d[score][0]!s}"

# Each method works equally well!
print( debug_string1.format( **counter_dict ))
print( debug_string2.format( d = counter_dict ))
```

```
'CounterDict' has 5 counts and a score of 6.4
'CounterDict' has 5 counts and a score of 6.4
```


What's more, the new way of string formatting also lets you use **_objects!_**

```python
class Counter(object):
    title = "CounterObj"
    counter = 5
    score = (6.4, 3)
debug_string3 = "'{o.title}' has {o.counter!s} counts and a score of {o.score[0]!s}"
print( debug_string3.format( o = Counter() ))
```

```
'CounterObj' has 5 counts and a score of 6.4
```

### Absurd amounts of control

You can pad numbers (e.g., to have at least 6 digits with 2 after the decimal point: `'{:06.2f}'.format(3.141592653589793)`), you can pad strings with characters (`'{:_<10}'.format('test')`), or you can even get crazier and center-align strings with padding (`'{:^10}'.format('test')`).

I sometimes like to have lines that break up code or logs into sections. Now I make those lines standardized for width and super pretty: 

```python
section_header = "# {section_name:-^80s} #"
# Or make the padding character and total width variable!
custom_header = "# {section_name:{pad_char}^{total_width}s} #"

logging.info(custom_header.format(section_name = "Booting up!",
                                  pad_char = "=", total_width=40 )) 
                                  
# Logs: '# ==============Booting up!=============== #'
```

You see that with the new formatting methods, you can dynamically specify the parameters of the formatting with _other_ variables by embedding curly braces in curly braces.

Cray!



<hr />
<br />

## Source Code:

> [`scanlation_scraper_timed.py`](https://github.com/burchill/burchill.github.io/blob/master/code/4chan-image-scraper/scanlation_scraper_timed.py)

The end-product of my pains. I gave up adding doc strings halfway, but I have a lot of comments, so understanding what's happening shouldn't be too hard. I made the argument-parsing nice and sexy--try `python3 scanlation_scraper_timed.py -h` for a look-see. 

For an idea of how threads work, maybe check out my [previous post]({{ site.baseurl }}{% post_url 2016-11-14-webcomic_post %}) about scraping with threads.
