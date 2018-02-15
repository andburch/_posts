---
layout:  post
title: "Python Lessons from 4chan, Part I: Logging is Easy!"
comments:  true
published:  true
author: "Zach Burchill"
date: 2018-02-10 10:00:00
permalink: /fourchan_logging/
categories: ["python lessons from 4chan",python,'web scraping','4chan','manga','webcomics',logging]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---



If I wanted to make this post sound professional and industrious, I say that my motivations behind this project were because I've started working towards my Bayesian model of webcomic updates again, and I'm taking an intermediate step by analyzing data from similar content creators.

But the truth is, I was just pissed off that I couldn't read the manga I wanted to.

These are the Python lessons I learned scraping manga scanlations off of 4chan.

### Part 1: Logging is easy!

<!--more-->

## Background

I understand that no matter what I say, writing a post about _[manga](https://en.wikipedia.org/wiki/Manga)_ (Japanese comics) right after a post about [traditional Japanese aesthetics]({{ site.baseurl }}{% post_url 2017-11-16-nightmoves %}) makes me seem like a weeaboo. But _dammit, I'm not!_ I just like reading comics online, so my college roommate showed me a comics site that hosted a [grim, murder-mystery remake of _Astro Boy_](https://en.wikipedia.org/wiki/Pluto_(manga)) and a [comedy about a superhero bored with how strong he is](https://en.wikipedia.org/wiki/One-Punch_Man). 

I wanted to read the latest chapters, so I ended up on a manga message board to find them... So I saw more manga that looked interesting and so I started reading more... So I started spending more time on the message board... So... now I'm reading six Japanese rom-coms concurrently, and really into [a group of adventurers making food in a dungeon](https://en.wikipedia.org/wiki/Delicious_in_Dungeon) and [the fallout of the Russo-Japanese war on Japan's indigenous peoples](https://en.wikipedia.org/wiki/Golden_Kamuy). 

Still not a weeaboo, though.

## My Problem

Japanese comics--perhaps unsurprisingly--are mostly written in Japanese. Since many are never officially published in English, there are groups of enthusiasts who scan the ones they like, translate them, and put them on the internet for free. These are called _**scanlations**_, and they're _kinda_ [classified as pirating](https://en.wikipedia.org/wiki/Scanlation#Legal_action).

The decentralized, volunteered origins of these translations makes keeping track of them somewhat difficult, and I frequently find that chapters are missing online, or have messed up ordering, etc. Many scanlations come from 4chan's anime/manga message board, which means that if no one re-uploads them to a separate hosting site, they eventually get deleted or end up lost in the annals of some third-party archiver.

### Warning: never, _ever_ go to 4chan.

I cannot stress this enough: **Mom, if you're reading this, do _not_ go to 4chan**.

If you are a loved one, or have any flicker of warmth in your heart, _**do NOT go to 4chan**_.  It's like the internet's unconscious id, vomitting out content that no decent human should ever see. The anime/manga message board (`/a/`), is _much_ tamer than the disturbing "random" board (`/b/`) or the infamously fascist/far-right "politics" board (`/pol/`), but even then, it's still really... _inappropriate_. I'm just there for translations of a few series that I can't get elsewhere.

I wanted to make sure that I didn't miss any updates, so I decided to write code that would **run** in the background of my computer, **check** to see if new translations had been released, and then **save** all the images to my computer. I wanted to do this: 

1. using Python's `threading` module[^1], 

2. as a way to try out some new techniques,

3. and in such a way that it would be _relatively_ easy for Python newbies to understand.

Long story short, this was _much_ more irritating than I thought it was going to be, and I wanted to share some of what I learned so that others can avoid my pain.

In order to avoid a insanely long post, I've broken my take-aways into a few shorter posts.  This is the first one: 

## Logging errors isn't that scary 

Since high school, a lot of the code I've written has been for _me_, meaning that only I have to understand it. Doing collaborative research in a computer science lab, developing helpful R packages, and making shareable scientific code has meant that I've needed to change the ways I used to work, but I still struggle with my natural laziness.

Since I was making shareable code that would be running in the background, I was going to need to do more than just `print("THIS BROKE")`. I was going to have to **log** the scraper's activity to a file. Luckily, Python has a module that makes this pretty dang easy: `logging`.

The docs have a [pretty decent tutorial](https://docs.python.org/3/howto/logging.html#logging-basic-tutorial), but for the simplest cases, all you basically need to do is to set up the logging configurations:

```python
import logging

logging.basicConfig(filename = "main.log", 
                    level = logging.DEBUG, 
                    # preface each log with time, thread name, and priority level
                    format='%(asctime)s -- %(threadName)s %(levelname)s %(message)s')
```

And you can start logging information at different levels of importance:
                    
```python
logging.info("Starting up!")
logging.warning("Uh oh, something bad happened!")

try:
    logging.debug("In the try statement")
    x=0/0
except ZeroDivisionError:
    logging.exception("Oops, captured an exception. Here's the trace!")
```

Note that if you change the level in `logging.basicConfig`, you can decide what levels of priority you want actually logged. If the level is set to `logging.WARNING`, only messages with priority levels at "warning" or above (i.e., errors) will be logged, etc.

It's just _that_ simple!

<hr />
<br />

## Source Code:

> [`scanlation_scraper_timed.py`]({{ site.url }}/code/4chan-image-scraper/scanlation_scraper_timed.py)

The end-product of my pains. I gave up adding doc strings halfway, but I have a lot of comments, so understanding what's happening shouldn't be too hard. I made the argument-parsing nice and sexy--try `python3 scanlation_scraper_timed.py -h` for a look-see. 

For an idea of how threads work, maybe check out my [previous post]({{ site.baseurl }}{% post_url 2016-11-14-webcomic_post %}) about scraping with threads.


### Footnotes:

[^1]: Just to be clear, we doin' Python 3, baby. None of that _old_ crap.


