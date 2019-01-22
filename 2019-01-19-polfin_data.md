---
layout:  post
title: "Why Political Finance Data Needs Open Source"
comments:  true
published:  true
author: "Zach Burchill"
date: 2019-01-19 10:00:00
permalink: /polfin_data/
categories: [R,'web scraping',politics,finance,data,opensecrets,"Center for Responsive Politics","open source",essay]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---



Did you know that information on the personal finances of anyone running for US Congress, the Presidency, and any number of federal offices is [available to the public](https://ballotpedia.org/Household_net_worth_(Member_of_Congress))? It's true, and an organization has collected all this information, processed it, and put into a single format and location for free!  A perfect treasure trove for data scientists and researchers!

The only catch is... _it sucks_.

<!--more-->

Well, [OpenSecrets.org](https://www.opensecrets.org/) doesn't exactly "suck", but it has some relatively fatal flaws. In this post I describe my frustration with OpenSecrets and why their goals would be better served by moving their project to full open source.

## Background

A few years ago, an event happened that made me realize that _anyone_ can be successful in politics. 

In the following years, a near _constant_ string of events have made me realize that I could probably make better decisions than many of these elected officials.  But when I compare myself to successful politicians, I notice there's one thing they have that I don't: **cash**.

As a researcher, scientist, and data scientist, I wanted to ask a question: _how much money do you need to get elected?_  I came into this area knowing literally nothing about political finances, how to get the data, or what I should have expected---I just wanted to ask a few questions and see what the data suggested.  What followed was a series of yo-yo-like ups and downs, alternating between being hopeful and impressed to disappointed and jaded. 

## OpenSecrets.org Data

I'll preface this with the following: *if you know a better place to get personal financial disclosure (PFD) data, please leave a comment below.* I could have very well spent a lot of time looking in the wrong places. But here's a very shortened summary of what I went through:

<aside class="right"><p>OpenSecrets pro tip:</p><p>If you're going to analyze the OpenSecrets data, use their <a href="https://www.opensecrets.org/bulk-data/downloads">bulk data</a> and download it all at one time. Don't waste your time with their API.</p></aside> 

OpenSecrets.org, the website for the [Center for Responsive Politics](https://en.wikipedia.org/wiki/Center_for_Responsive_Politics) is a website dedicated to tracking money in politics.  If you're looking for free comprehensive data on politicians' personal or campaign finances, they're just about the only shop in town. The idea behind the organization's work---helping reporters, researchers, and policy-makers sort through lobbying money---is brilliant and much-needed.  However, as with so many amazing projects for the public's good, I'm pretty sure they're also understaffed and underfunded.

Although their website is well organized and very user-friendly, it really only seems useful for giving higher-level summaries of politicians' funding and assets. To really do anything right---whether it's investigative journalism or thorough analyses, you need the actual data. And let me tell you, their _actual_ data is anything but well organized and user-friendly.

### Criticism #1: No effort is spent making data readable

Here's the thing: I know OpenSecrets has their data stored in a useable form for the backend of their website, given their API. I also know that pretty much _any_ format of that database, it should be relatively trivial to output that data into pretty much any flavor of tab-delimited file. So why do they put it in the weirdest, jankiest format possible?  It's technically a comma-separated value file, but strings are quoted by "\|", nothing is escaped _in_ those strings, and there are some rows that span multiple lines.

<aside class="right"><p>R CSV readers have weird limitations</p><p>Evidently R's <code class="highlighter-rouge">read.csv</code>, <code class="highlighter-rouge">read_csv</code>, and their ilk can't handle escaped quotes! Seems like a relatively big problem, but whatever! I ended up using <code class="highlighter-rouge">data.table::fread</code> to read the OpenSecrets data.</p></aside>

The choice of quoting character is odd, but not "sinful".  The fact that they don't escape anything _is_. It just so happens that if you break things up by "\|,",[^2] it will "mostly" work, But the second a politician uses "\|," as a string in any descriptions of their assets, all their data will fail to be machine-readable. And there are hundreds of no-effort libraries that can escape special characters, so their choice _not_ to is baffling.  

Although it probably would have been easier to read via Python, I wanted to use R. I ended up using Python to convert it to something R could read with what ended up being just some regex.[^3] And even then, I think there were still a few cases I had to correct by hand.

### Criticism #2: No "unit" testing, no standardization, no checking

My realization that maybe the CRP was underfunded first started dawning on me when I was reading [their documentation](https://www.opensecrets.org/resources/datadictionary/Data%20Dictionary%20pfd_income.htm) and regularly found spelling errors and typos. "Rerfers" for "refers", "subsiquent" for "subsequent", "is filer jointly with spouse" for "is file[d] jointly with spouse", etc.  These are mistakes that basic spell-checking should catch---what were they doing in so many "official" docs?

But the data has many problems that should have been caught by even _semi_-diligent sanity checks. There are small, relatively unimportant things like inconsistent capitalization of coding values and whether NAs are represented by `" "` or `""`, etc. But there are also bigger issues.

For example, OpenSecrets doesn't have a winner for [these](https://ballotpedia.org/Washington%27s_4th_Congressional_District_elections,_2014) [three](https://ballotpedia.org/United_States_House_of_Representatives_election_in_the_Northern_Mariana_Islands,_2014) [races](https://ballotpedia.org/Louisiana%27s_5th_Congressional_District_elections,_2014) in 2014, and a bunch of other years. Or, according to the docs, there are only supposed to be four types of PFD reports, represented by "Y", "A", "N", or "T". So what the heck do the 124 entries that are marked "C" or "O" in the income data mean?

<aside class="left"><p>Fun fact:</p><p>While testing ways to connect my scraped data with the CRP data, I ended up learning of <a href="https://www.alternet.org/2017/10/two-senate-candidates-busted-using-aliases-are-possibly-married-each-other-no-one-can-figure-it-out/">a mysterious husband and wife pair</a> who ran as opposing Democrats / Republicans in Montana's Senate race, only to <a href="https://www.havredailynews.com/story/2018/03/13/local/deans-disappear-from-senate-race/518123.html">disappear just as mysteriously</a>.</p></aside>
 

It would be _trivial_ to write code that ensures that the definitions outlined in the documentation actually apply to the data. Keeping the definitions and these tests together would also ensure that whenever the definitions were updated, the docs would be too. These basic sanity checks could also be easily expanded to other sorts of tests. For example, we _know_ that each race needs to have exactly one winner. It took me about three lines of code to check for that.  If, by some act of God, this wasn't _always_ true, it's so _vastly_ true most of the time that we'd want to know what the exceptions are anyway. Or, for example, you can easily check to make sure any candidate who won must be in a primary,[^4] or can't run twice in the same primary for the same position. These problems could in theory be caught very easily, but there are other issues with the data that seem more insidious.


### Criticism #3: Lack of transparency -- weird data

Although the Center for Responsive Politics aims to increase transparency in politics, its own methods are relatively opaque. In theory, this doesn't have to be a problem if the organization always did everything _right_, but we all know that big projects like these invariably have screw-ups, and without transparency, these possible mistakes become quagmires for anyone using the data.

When trying to decipher the "C" and "O" PFD report types, I stumbled across data that, at first, just seemed _wrong_. For example, in the income data file, `PFDincome.txt`, the row with `ID` `Z140277592`. It is purportedly from a 2014 filing from Trent Kelly, a Representative from Mississippi, and says [his wife was paid $15,117 by "Grammer Inc."](https://www.opensecrets.org/personal-finances/other-data/Trent-Kelly?cid=N00037003&year=2014) The problem is, as far as I can tell, Kelly never _filed_ in 2014, and none of his later filings mention Grammer Inc.

<aside class="left"><p>Fun fact:</p><p>Only two Senate candidates had personal finances so <i>vast</i> that they timed out my web-scraper. Which two? </p><p>Mitt Romney and Scott Walker!</p></aside>

**_However_**, I did some sleuthing and found that his wife, Sheila Kelly, _does_ work for a Grammer Inc. _Further_ sleuthing (i.e., last-minute fact-checking) found that Kelly actually filed _twice_ in 2015, once as an admitted member of the House of Representatives, and once as a _candidate_. I found that [his _candidate_ filing](http://clerk.house.gov/public_disc/financial-pdfs/2015/10011514.pdf) has data that matches the data that OpenSecrets evidently uses. But the data OpenSecrets uses is clearly meant for 2015. But perhaps the "C" report types are from "candidate" reports? So that... _almost_ makes sense---it would seem like they just got the date wrong! 

Sadly, no. First, there are essentially only 33 unique "C" reports in the assets data, dating back to 2009. [Over 200 incumbents were defeated in the House in 2016](https://ballotpedia.org/Incumbents_defeated_in_2016%27s_state_legislative_elections) alone, so there should be _much_ more candidate entries than that.  Secondly, OpenSecrets does not have any PFD data on Rep. Kelly after "2014", despite him _still being a House Rep!_  This issue connects back to the need for testing, but it seems _insane_ that OpenSecrets doesn't check for current Congressmen with _no_ information. Perhaps you can see why I stopped trusting their data.

### Criticism #4: Lack of transparency -- literally giving up

Did you know that since 2014, the Center for Responsive Politics no longer keeps track of PFDs that are filed via paper, despite the fact that they are available online? Neither did I!  I only found out accidentally!

I learned that the CRP seems to have stopped paying people for manual data entry for PFDs in 2014 after noticing [Dianne Feinstein](https://en.wikipedia.org/wiki/Dianne_Feinstein) doesn't have any data past that point. It would be odd for an established member of Congress to just _forget_ about filing, so I checked the Senate website and she _has_ been filing, just non-electronically. The scans of her paper forms are available online as `.pdf` files.

However, most other Senators and canidates file electronically, which means their PFDs are available on the Senate website as easily parsable HTML data (which I have since scraped myself). Following a hunch that maybe the reason the CRP doesn't have any of her data is because of the format, I decided to lean hard into the stereotype that old people don't know how the internet works. After all, Dianne Feinstein _is_ the oldest current Senator, so if she resisted the new electronic filing, maybe some of the other elderly Senators had too.

<aside class="left"><p>Political pro tip:</p><p>File all your PFDs via paper so it's harder for people to track your spending. The pain of manual data entry will make your crimes <i>that</i> much harder to spot!</p></aside>


My stereotype wasn't totally correct, but I discovered similar gaps for James Inhofe, Chris Van Hollen, and Tom Cotton, who all file non-electronically as well. It would seem that the CRP has set up some sort of web-scraper for the Senate website to automatically grab the PFD data, but has given up keeping anything that isn't already machine-readable. Which would be **_bonkers_** unless the organization is in its death throes funding-wise. Some of the most powerful members of Congress basically have their personal assets become invisible!

But it would be even **_more_** bonkers to make that policy change without telling anybody, especially the people trusting your data. And as far as I can tell, that's exactly what they did.

All these problems stem, in my mind, from one single issue that is crippling OpenSecrets. Luckily, there's a single fix for everything I've mentioned.

## Solution: Go open source

<aside class="right">
<p>Other sources of financial information:</p>
<p> * <a href="http://clerk.house.gov/public_disc/financial-search.aspx">US House of Representatives</a> <br />
 * <a href="https://efdsearch.senate.gov/search/home">US Senate</a>  <br />
 * <a href="https://www.fec.gov/data/browse-data/?tab=candidates">FEC campaign finances</a>
  </p>
</aside>

For OpenSecrets to be viable---or for any organization that wants to do something similar---it's clear to me that the way forward is to go full open source, at least for the data collection and curation. I can imagine this happening in a GitHub-style collaborative environment very easily---comparable to RStudio's [open](https://github.com/r-lib/rlang/) [source](https://github.com/tidyverse/dplyr) [projects](https://www.rstudio.com/products/rpackages/).

Not only does open source better fit with CRP's mission of "champion[ing] transparency", but going open source will let it actually _achieve_ its goal to "produce and disseminate peerless data [...] on money in politics".

### Making do with less funding

Let's face it: most non-profits are not exactly _rolling_ in cash.  The CRP has a limited budget, and I'm guessing a lot of that has to go to other things. 

Open source projects have a history of getting things done with _much_ less funding than similar non-open projects---they let people contribute for free!  I've always thought of open source projects as being fueled by "nerd passion", but imagine combining that passion with people's desire to hold their government accountable. We're talking _rocket fuel_ here!

### The use-contribute cycle

No one knows the problems of a system better than the people trying to use it, and those are the ones who want to improve it the most. It's in a project's best introduce to reduce the barrier to contributing as much as possible. Imagine if the OpenSecrets project was one GitHub! You'd have pull requests correcting the typos, improving the docs, and standardizing the data in a blink of an eye![^5]  

In fact, people would probably be willing to manually input (and double check) any data that couldn't be scraped, such as Senator Feinstein's PFDs, for *free*. People would also be able to see and propose new sanity checks to be run automatically so that data collection continually grows more robust, making the project more self-sufficient as time goes on!

### _Real_  transparency 

But perhaps the most important aspect of becoming open source is that you would be making it possible for other people to actually understand and trust the data. A GitHub-like platform would automatically keep track of changes, discussions, and explanations for why things are the way they are, but you could also envision major changes being explained in something like "releases" as well, similar to the `NEWS` file in R packages. Claims of partisanship would be hard to make when all decisions about the data are visible and above-board! 

Even if the data collection was somehow limited, users should _know_ those limitations---if there were any errors in the data, researchers could track them down and isolate their source, rather than having to treat _all_ the data as dubious.


## Conclusion

I hope OpenSecrets.org can improve, and if they aren't willing to, I think it's high time for a _real_ open source project to step in and take up their mantle. I've learned a lot working with this data, but I ultimately decided that the OpenSecrets data couldn't fit my needs regardless of the flaws I've outlined above: I wanted to compare the finances of those who lost and those who won their races, and the CRP doesn't store candidates' finances past the election unless they win, which makes sense given limited space.

I ended up scraping the data myself from the Senate's website. I wanted to try out `pandas`, so I made some Python classes that store the report data in `pandas` data tables---it worked pretty well! I've saved the information, but I've since realized that using Senate race data for my comparisons doesn't exactly make sense---the incumbent advantage seems strong enough to wipe out any effects of personal finance, and there were only 25 open seats in 2018 anyway.  If I get the time, I'll try to get data from the House, but that requires some `.pdf` conversion that I'm not really looking forward to...


<hr />
<br />

## Source Code:

> [`opensecrets_data_cleaning.R`](https://github.com/burchill/burchill.github.io/blob/master/code/political-finance/opensecrets_data_cleaning.R)

This is the code I used to do higher-level cleaning and organizing for some of the OpenSecrets PFD data. You can download the data yourself [here](https://www.opensecrets.org/bulk-data/downloads) for free if you're not using the data for commercial purposes.


### Footnotes:

[^1]: This becomes a running theme, FYI.

[^2]: It's actually more complicated than that, but that solves most of the problems.

[^3]: It ended up being this whopper: `re.sub(r"(?<!,)(?<!^)(?<!\\)(?<!\n)\|(?!(,|$|\n))", "\\|", ...)`

[^4]: OpenSecrets has Michele Bachmann in 2012 being the only candidate to do so in the years 2012-2018, so I'm assuming being able to do is a mistake in how they coded the data.

[^5]: I actually reported these problems to them via email, hoping to improve the project, but I was met with an auto-reply basically saying "we're too busy, we might get to it eventually". Open source management systems like those on GitHub would make managing these requests a breeze, comparatively.
