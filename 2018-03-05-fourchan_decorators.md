---
layout:  post
title: "Python Lessons from 4chan, Part 3: Are Decorators Pythonic?"
comments:  true
published:  true
author: "Zach Burchill"
date: 2018-03-05 10:00:00
permalink: /fourchan_decorators/
categories: ["python lessons from 4chan",python,'web scraping','4chan','manga','webcomics',"python decorators",decorators,pythonic]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---



If I wanted to make this post sound professional and industrious, I would say that my motivations behind this project were because I've started working towards my Bayesian model of webcomic updates again, and that I'm taking an intermediate step by analyzing data from similar content creators.

But the truth is, I was just pissed off that I couldn't read the manga I wanted to.

These are the Python lessons I learned scraping manga scanlations off of 4chan.

### Part 3: The 'pythonicity' of decorators

<!--more-->

_For the background on the project this post stems from, as well as other life lessons I learned, check out [Part 1]({{ site.baseurl }}{% post_url 2018-02-10-fourchan_logging %}), [Part 2]({{ site.baseurl }}{% post_url 2018-02-15-fourchan_strings %})._

## Getting Fancy

The drive for shiny new toys is not limited to children: statisticians and coders alike still get excited breaking in fancy new statistical techniques or that pretty new Python library. I myself feel the pull all the time: one third curiosity, one third actual utility, and
one third "keeping up with the Joneses." Mix well, serve on the rocks.

In working on this 4chan-scraping project--and discussing code architecture with a friend--I rediscovered something in Python that fit the "shiny toy" niche--something called **_decorators._**[^1]

### Decorators

You know how things like strings, ints, and objects can be saved as variables, passed as arguments into functions, or be returned?  Well in Python, functions _themselves_ can be treated similarly. A "decorator" is just a function that takes in a _different_ function as an argument and builds on that different function without modifying it.  Consider the following:

```python
def a_little_bit_louder_now():
    return("A little bit louder now")
def throw_my_hands_back():
    return("Throw my hands back")

def shout_decorator(a_function):
    def wrapper():
        prefix = "Shout! "
        message = a_function()
        suffix = "!"
        return(prefix + message + suffix)
    
    return(wrapper)
```

You can see that the function, `shout_decorator()` defines a function _within_ itself and then returns that function. You can also see that it takes in a function as an _argument_ and uses that function _in_ the one it returns.  So let's see what happens when we use the decorator on the functions we've made:

```python
kingsmen_verse_function  = shout_decorator(throw_my_hands_back)
kingsmen_chorus_function = shout_decorator(a_little_bit_louder_now)

print(kingsmen_verse_function())
print(kingsmen_chorus_function())
print(kingsmen_chorus_function())
```

```
Shout! Throw my hands back!
Shout! A little bit louder now!
Shout! A little bit louder now!
```

**Hey-ey, hey-ey!** Note that _unlike_ in the example above, "real" decorators are almost always used to _overwrite_ the original function they build on, like so:

```python
throw_my_hands_back  = shout_decorator(throw_my_hands_back)
a_little_bit_louder_now = shout_decorator(a_little_bit_louder_now)
```

### "Pie" syntax

The whole reason I started learning about decorators is because I saw an `@` symbol before a function and I wanted to learn what it meant. This `@` is because of Python's "pie syntax", a way of making it easier to read and write decorators.

After defining a decorator function, you can decorate the subsequent function by putting `@<decorator_function>` before you define it. The example below decorates `hey_ey()` with `quadruple_repeat_decorator()`.

```python
def quadruple_repeat_decorator(a_function):
    def wrapper():
        l=[]
        for i in range(4):
            l += [a_function]
        return( " ".join(l) )
    
    return(wrapper)
    
@quadruple_repeat_decorator
def hey_ey():
    return("Hey-ey!")
```

If you want to make even more generalizable code, you can make decorator functions that take in other arguments as well. For example, if you want a decorator that will join the output of a function an _arbitrary_ amount of times, you can do so:

```python
def repeat_n_times(n):
    def repeat_decorator(a_function):
        def wrapper():
            l=[]
            for i in range(n):
                l += [a_function()]
            return( " ".join(l) )
    
        return(wrapper)
    return(repeat_decorator)
    
@repeat_n_times(4)
def hey_ey():
    return("Hey-ey!")
 
@repeat_n_times(2)
def a_little_bit_softer():
    return("Shout! A little bit softer now!")   
    
print(hey_ey())
print(a_little_bit_softer())
```

```
Hey-ey! Hey-ey! Hey-ey! Hey-ey!
Shout! A little bit softer now! Shout! A little bit softer now!
```

This is super useful for a lot of specific situations. For example, `Flask` is a Python library used for building websites, and [uses decorators frequently and elegantly](http://flask.pocoo.org/docs/0.10/quickstart/#routing) to set URLs. Decorators are also great for timing functions and certain [other advanced applications](https://wiki.python.org/moin/PythonDecoratorLibrary).

However, as they get more complicated, or as you chain decorators together, they get _much_ harder to read, especially for those who haven't worked in all the obscure parts of Python. Feel free to browse through some of the examples---they're often a little mind-bending.

## Sometimes Showing Off Is a Bad Thing

I have a function in my code that attempts to load information from a URL `n` times in a row, waiting `m` seconds each attempt, catching certain exceptions, etc. It just so happened that I can use that one function to both load HTML pages as well as images, but it might be nice to have a decorator that will do the same for an arbitrary function.  It would have also been a great instance where I could show of my skills.

There are obvious benefits to using decorators in this situation, and maybe if I were developing an entire framework of code, I would use the decorator version. But stackoverflow user Kevin J. Rice [points out something worth considering](https://stackoverflow.com/questions/489720/what-are-some-common-uses-for-python-decorators#comment42762992_490090):

> Decorators are inherently confusing, esp. to first-year noobs who come behind you and try to mod your code. [... They're] vastly overused by people wanting to seem smart (and many actually are) but then the code comes to mere mortals and gets effed-up.

> [C]ode readability is just about my highest priority when writing. Code is read 7+ times for every time it's modified. Hard to understand code (for noobs or for experts who are working under time pressure) is technical debt that has to be paid every time someone visits the source tree

Considering his points, as well as the fact that I am purposefully making code that aims to be more accessible to newbies, I decided against using decorators. In my opinion, the guiding principles of Python suggest that I shouldn't go to town with decorators on this project as much as I had originally intended.

From [The Zen of Python](http://legacy.python.org/dev/peps/pep-0020/) by Tim Peters:

> Beautiful is better than ugly. <br />Explicit is better than implicit.<br />Simple is better than complex [...] <br />If the implementation is hard to explain, it's a bad idea. <br />If the implementation is easy to explain, it may be a good idea.

Although decorators are important to some types of projects, I don't think it would have been "pythonic" to use them here, or to shoe-horn them in other parts of the code that I was writing. Sometimes it's better to keep things simple than make them efficient.


<hr />
<br />

## Source Code:

> [`scanlation_scraper_timed.py`]({{ site.url }}/code/4chan-image-scraper/scanlation_scraper_timed.py)

The end-product of my pains. I gave up adding doc strings halfway, but I have a lot of comments, so understanding what's happening shouldn't be too hard. I made the argument-parsing nice and sexy--try `python3 scanlation_scraper_timed.py -h` for a look-see. 

For an idea of how threads work, maybe check out my [previous post]({{ site.baseurl }}{% post_url 2016-11-14-webcomic_post %}) about scraping with threads.

### Footnotes:

[^1]: A better blogger than me has written a very good introduction to the concept [here](https://realpython.com/blog/python/primer-on-python-decorators/), which I definitely recommend reading in more depth if you personally want to use them. However, I'll mostly be talking about when _not_ to use them.
