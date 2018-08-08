---
layout:  post
title: "My Problems (and Solutions to) Scraping LOTS of Data"
comments:  true
published:  true
author: "Zach Burchill"
date: 2018-08-08 08:00:00
permalink: /scraping_problems/
categories: ["python lessons from 4chan",python,'web scraping','manga','webcomics',"python library",requests,urllib,shelve,decorators]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---



Did you know that if you try to scrape too many pages at a time from the same website, it will sometimes think you're being malicious and block your IP address?  Or that using Python's `urllib` or `shelve` packages totally sucks on some computers?

Come, let me <span style="text-decoration: line-through;">show off</span> teach you about some of the more nuanced aspects of web-scraping buttloads of data from the internet.

<!--more-->

## Background

First off, let me just say that although a _lot_ of my posts on this blog are about web-scraping, I'm not an expert in it, and I primarily use it to get data that I find interesting. In [previous]({{ site.baseurl }}{% post_url 2016-11-14-webcomic_post %}) [posts]({{ site.baseurl }}{% post_url 2018-02-10-fourchan_logging %}), I've talked about how I'm interested in making a statistical model that can predict when webcomics will likely stop updating---this post comes from my attempts to analyze similar-but-more-accessible data as a stepping stone to that greater goal.

### Scanlation records online

There just so happens to be a very comprehensive database of manga scanlations online---not one that actually hosts or links to the scanlations, but that records information about the series and when they have new scanlations released. (For those of you who don't know what _scanlations_ are, see my [previous posts]({{ site.baseurl }}{% post_url 2018-02-10-fourchan_logging %}) as an intro.)  

It has logs of updates for hundreds of _thousands_ of different Japanese comics which seems like it would be great for more complex statistical modeling.  Although there are important differences between how these comics are released and how webcomics are updated (which I will deal with in my future analysis), the similarities between the two make them seem relatively comparable.

Unfortunately, there was no "download all"" button.

### My goal: scrape the relevant data and save it locally as a database

In trying to download all the relevant information, I ran into problems almost immediately.  The three most interesting / useful things I learned are as follows:

1. For anything using HTTP requests, always use the `requests` Python package instead of `urllib` or any of its descendants.

2. For serious web-scraping, you might have to use proxies to not get blocked. I came up with a relatively cool way to deal with this issue.

3. The Python `shelve` module _sucks_ on some set-ups, and unless you _really_ need to `pickle` things, there are much better options to save basic database information.

## 1: Use the 'requests' library

This post is already too long, so I'm going to keep it simple for this piece of advice. Kenny Myers puts it best on the website for the [`requests` library](http://docs.python-requests.org/en/master/): 

> Python HTTP: When in doubt, or when not in doubt, use `Requests`. Beautiful, simple, Pythonic.

That's it, really. If you're dealing with HTTP requests (i.e., almost all of web-scraping), instead of trying to hash something out with Python's `urllib`, `urllib2`, or even the independent `urllib3`, just use the amazing [`requests` library](http://docs.python-requests.org/en/master/). Please, just save yourself so many headaches and sweat and blood and tears and all that.  It's just literally the best.


### Python libraries and getting internet data

For all the web-scraping code I'd ever written in Python, I had used the standard Python libraries to connect to the internet.  We're talking about that sweet, sweet `urllib`/`urllib2` action. I would load pages like:

```python
import urllib
from bs4 import BeautifulSoup

url = "http://boards.4chan.org/a/catalog/"
r = urllib.request.urlopen(url).read()
soup = BeautifulSoup(r)
```
Sometimes it wouldn't work. After debugging and Googling, I found most of these cases to be relatively reasonable things I needed to account for, like websites getting pissed when you give them a weird browser header. 

But many screw-ups seemed ephemeral---I could run the same code twice and not get the same error.  I chalked this up to my Wi-Fi connection and the fickle beast we call the internet.  About half of these issues went away when I switched to `requests`.

And I don't mean that you shouldn't ever use _any_ of the functions in `urllib<N>` libraries---but for the code that's actually doing the _requesting_, use `requests`.  Every time I dig into the docs and tutorials, I'm impressed anew.  I'll be demonstrating how to use some of the proxy features in the next section, and in a (possible) future post, I'll demonstrate how I used it to get through a Javascript password prompt on my friend's website.[^1]

I solved the other half of the problems when I realized my computer was being mistaken for a node in a DDoS attack.

## Using proxies

### Getting blocked

When I first started trying to scrape information on thousands of manga series, I noticed something weird: after a couple of minutes, I would start getting more and more HTTP requests that would fail.

Curious, I tried to debug by documenting how many requests would fail (from a subset of the total requests I intended) as I systematically changed parts of the code, rerunning it every time.  I did this a few times before _every_ request started to fail. Perplexed, I tried connecting to the site in a normal browser.

Nope. It looked like it was down. But little ole suspicious me, I checked whether it was _really_ down via [isitdownrightnow.com](http://www.isitdownrightnow.com/). Evidently it was up for everybody else, but not for me. Confirming this via a few in-browser proxies, I finally realized that they had _blocked_ me.

_They_ had blocked _me_.

Yeah, I mean, _yeah_, that technically "makes" "sense." They probably have some kind of automated system that assumed I was part of an emerging [DDoS attack](https://en.wikipedia.org/wiki/Denial-of-service_attack).  I was only using ~100 threads to handle requests over my crappy internet connection, but I guess that was enough.  

### Getting around it

There are free-to-use proxies on the internet, that let you in essence mask your IP address.  I found that the proxies at [sslproxies.org](https://www.sslproxies.org/) are _insanely_ easy to use with Python's `requests` package.[^2]

Basically, I web-scraped the proxy IP addresses and ports from sslproxies.org, and plopped them into `requests.get`:

```python
proxy_d = { "https": "http://{ip}:{port}".format(ip = proxy_ip, port = proxy_port) }
r = requests.get(url, proxy = proxy_d)
```

Kabloom.  Easy as pie.

The only problem is that I'm making thousands of requests--even if I have a proxy, they'll probably just block it after a few hundred uses.  The idea situation would be to cycle through multiple proxies as each is blocked, or use each only a few times before moving on to the next.  But my whole scraping system is designed for completely parallel and asynchronous threads, so I can't keep track of how many times a request has been made with something like a `for` loop.

### Return of the decorators

Remember how in an earlier post [I questioned how "pythonic" decorators really were]({{ site.baseurl }}{% post_url 2018-03-05-fourchan_decorators %})?  Well, I kind of went crazy with them for this project.

Rather than defend myself, I'll just say that yes, there _are_ times when decorators actually are very helpful and good, such as when you need to ["memoize" things](https://en.wikipedia.org/wiki/Memoization) or---as was the case here---to keep track of how many times a function has been called.

I used decorators here to turn the functions I used to call requests into objects that kept track of how many requests had been made, and which automatically handled how proxies were loaded, used, and discarded.  I've called these functions/decorators "ninja" functions in the source code, as they help _sneak_ around some of overly harsh blocking techniques some sites have.

### "Ninja" decorators

I did this two ways: first, I made a more general, flexible code scheme of implementing these decorators (using `manage_proxies()` and `proxy_decorator()` in the source code), and then I said 'screw it' and made a simpler version that better fitted my current needs (which doesn't _really_ even use decorators, but employs a similar concept).

I actually made `manage_proxies()` a class that would be initialized with a given function, and use that function every time it was called, while counting, loading, and switching between proxies in the background.  Since decorators tend to only take in a single argument (i.e., the function in the line below them) when used with the `@` syntactic sugar, and because I wanted to let the number of requests attempted before switching between proxies be an argument, I made the `proxy_decorator()` function, which accomplishes that.

Confused?  Yeah, honestly, so am I---even though I wrote the code, I pretty quickly switched to the simpler version (`ninja_soupify_simpler`), and have forgotten a little bit of what these two functions did.

IIRC, you can turn `manage_proxies` into a decorator by just removing the `switch_after_n` argument in its `__init__`. I think I set it up the way it is due to some (possible) constraints in how syntactic sugar works.  Basically so you can do something like this, where `switch_after_n` is specified on the fly.

```python
@proxy_decorator(switch_after_n = 10)
def request_stuff(...)
    ...
```

As I said, in the end, I opted to make the code simpler and more compact by just making a single class called `ninja_soupify_simpler()`, which is less exciting but way easier to read and use.  Maybe this just further proves my point about decorators being confusing?

## Shelves suck

F\*\*\* shelves and f\*\*\* anyone who suggests using them on stackexchange. 

Remember how I wanted to take all this manga information and then _use_ it later?  That means I had to _save_ it, and this information was pretty 'rich'---it was a lot of nested information, like lists of authors/artists, user reviews/scoring, tags, etc. It seemed like simply saving this in a text file wasn't the right answer.

But hold up, before I start ranting about shelves, let me backtrack a bit: Actually, decorators are still useful! Or maybe not?

### Ok, decorators really _are_ useful sometimes?

Before I had worked everything out, I would frequently encounter issues/problems I hadn't anticipated.  I wanted my code to save its progress every so often to ensure that a crash wouldn't set me back to square one.  If I or something else stopped it while it was running, I wanted the information that it _did_ scrape to be safe.

Additionally, I didn't want my RAM to explode as I began storing ever greater dictionaries in memory---I wanted to "flush" what I had saved from the cache periodically, saving it to the hard drive instead.[^3] In order to keep track of how many series had been saved, I yet again turned to decorators!

The `save_progress` class in the source code is my crappy answer to that. Every time I successfully loaded a series' information, I would pass it to an instance of `save_progress` which would add it to a dictionary, and after every _n_ items, add the dictionary to a file saved on disk.  On one hand, using a class for each save file keeps everything tidy and in one place---the file being saved to, the functional equivalent of a [global interpreter lock](https://realpython.com/python-gil/), and the data being accumulated.  

On the other hand... wait. 

`save_progress` isn't _really_ (functionally) a decorator either, when I think about it. It's just a class that holds objects and saves them. F\*\*\* it, I'm not rewriting any of this. F\*\*\* decorators. F\*\*\* 'em.

### F\*\*\* `shelve` even harder though

Until recently, `save_progress` was even more of a hideous piece of sh\*\*.[^4] It had a large amount of seemingly redundant variables, and positively _spewed_ print statements. This was because the `shelve` module in Python sucks a\*\* and I needed to understand how.

Remember how we needed to _save_ things?  How do we save lots of Python data in big dictionaries?  Well, let's Google "save python information".

The first result for me is [this stackexchange post](https://stackoverflow.com/questions/17179222/python-saving-data-outside-of-program), which recommends using `shelve`.  Note, dear reader, the top-rated response from one "jabaldonedo" (emphasis added):

> Then you can recover your data **_<u>easily</u>_**

or how "ducin" responds:

> that's the best possible answer. If you need something basic, use shelve

Fools. Damnable fools.  There are many signposts on the internet that will point you towards using `shelve`, but do not heed their words, for they are from the devil.

### The madness begins

Shelves look great, for sure.  And they're so easy to implement!  I quickly hopped on board, whipped up some code, and got down to business.  Everything was great until my first "big" test run.

The first thing I noticed after saving ~1k entries was that when I tried to iterate over the keys or items in the loaded shelf, I would get the error:

```
SystemError: Negative size passed to PyBytes_FromStringAndSize
```

After the psychological equivalent of scraping my face against concrete for a _long_ time, I found that `shelve` was _saving_ my data, but it just wasn't letting me access the keys.  If I knew what the set of all possible keys was, I could just iterate over _those_, trying each one to see if it was in the dictionary and getting its value that way. Luckily, I had this set, so I could still access the saved data.

As far as I could deduce, my problems arose from something to do with my version of OSX not having/using something `gdbm`-related. And given [the state of my Python environment](https://xkcd.com/1987/), there was no way I was going to try reinstalling everything.

Whatever, it still worked, right?

### It didn't work

After my first "full" full run, I opened my saved `.db` file, which had grown to a whopping 170 MB.  Lo and behold, I could literally only access one value.

Trying to access every single possible key failed except for one.  A `shelve` object with that single key-value pair would be ~16 KB, so there _had_ to be more data in there but sadly, it was beyond mortal kenning.  Even attempting to open the `.db` file would show partial bits of plain text from other entries, but because it was pickled, it was unreadable on the whole.[^5]

After trying to add 90 _more_ entries (in a bizarre debugging attempt), I could access _three_ keys, but although the third key registered as being "in" the dictionary, trying to access its value returned:

```
_pickle.UnpicklingError: invalid load key, ''.
```

### _Just use JSON!_

To make a long story short, I ended up trying `json` and saving everything in a JSON file via a `shelve`-analogue recipe written by [Raymond Hettinger](https://code.activestate.com/recipes/576642/).  Not only were the resulting file sizes _much_ smaller, but the output is human-readable and can be easily read by R without having to go through Python.

## Coming soon

So I've got all the data now, but I realize that making a good model is going to be _way_ harder than I thought. For right now, I'll probably just start with exploring the data---visualizing it and looking at rudimentary patterns.


<hr />
<br />

## Source Code:

Check out the [Github repo for my current web-scraping endeavors](https://github.com/burchill/web-scraping). It's a little chaotic right now, so I'll highlight the most important files and briefly describe them.

> [`basic_functions.py`](https://github.com/burchill/web-scraping/blob/359097dc61dbf584e3363e38aebf7a886cd698ee/src/basic_functions.py)

This is probably the "cleanest" section of the code I've written. It contains the "project-neutral" functions I've written and expect to use repeatedly.  Pretty much all of the code that I've discussed in this post will be there.  The link above will take you to the version of the code at the time that I posted this. Check the most current repo if you want to see how/if I've changed it in the meantime. 

> [`manga_updates.py`](https://github.com/burchill/web-scraping/blob/359097dc61dbf584e3363e38aebf7a886cd698ee/src/manga_updates.py)

**Warning: this code is _highly_ idiosyncratic**. This is the the project-specific code I wrote to get the data that I did. I am _not_ going to explain it, but you can get an idea of how I actually used some of the functions in [`basic_functions.py`](https://github.com/burchill/web-scraping/blob/359097dc61dbf584e3363e38aebf7a886cd698ee/src/basic_functions.py). Basically, it has two functions: 1) it scrapes the "metadata" of the series I want (i.e., their "static" information, such as their descriptions and reviews), and 2) it scrapes the "issue" information of these series, getting all the times they were updated.

**_Please_ do not try to run `manga_updates.py` to get the data yourself.** See the [`README.md`](https://github.com/burchill/web-scraping) on the repo to understand why.



### Footnotes:

[^1]: Note: I don't mean using it to _hack_---I _knew_ the password because he gave it to me---I just mean using it to enter the password and store the cookies so it could access the rest of the site.

[^2]: I credit Danilo Polani in the source code, but I should here as well---his [blog post about rotating proxies](https://codelike.pro/create-a-crawler-with-rotating-ip-proxy-in-python/) as _super_ helpful in learning how to use them, and I based my asynchronous version on his code.

[^3]: In retrospect: 1) I'm not _100\%_ sure shelves remove the cache (although the docs say they do), and 2) looking back, I don't think I actually ever _needed_ to clear the cache for the amount of data I actually loaded, but I didn't know that going in. I think I honestly could have gotten away with a GIL and a simple dictionary, but I like what I have now.

[^4]: I only fixed it up because I realized there wasn't really an excuse to keep it like that after writing  this blog post.

[^5]: From the `shelve` docs: "The database is also (unfortunately) subject to the limitations of dbm, if it is used---this means that (the pickled representation of) the objects stored in the database should be fairly small, and in rare cases key collisions may cause the database to refuse updates.
