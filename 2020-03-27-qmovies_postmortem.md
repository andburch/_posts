---
layout:  post
title: "Hollywood Superstitions vs. Data Science: Post-mortem"
comments:  true
published:  true
author: "Zachary and Andrew Burchill"
date: 2020-03-26 00:30:00
permalink: /qmovies_postmortem/
categories: [R,IMDB,movies,"question marks","post-mortem"]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---

You maybe have seen the [post my brother and I just made about our investigation of some of the Hollywood superstitions]({{ site.baseurl }}{% post_url 2020-03-27-questionable_movies %}).  If not, go read that first.  This is just a little writeup about some of the technical problems we had working on the project, perfect for someone interested in how to do their own fast and dirty data science investigations, or someone interested in listening to me complain.

<!--more-->

When Andrew first came to me with [this project idea]({{ site.baseurl }}{% post_url 2020-03-27-questionable_movies %}), it sounded like a great, easy project.  My impression was that we could bang this out in an afternoon or two and be done with it.  All we had to *really* do was identify which movie titles were questions, and the rest would just be some basic data science.

Unfortunately, after the first few hacks I threw at the problem failed miserably,[^1] I realized that it wasn't going to be as simple as I first thought. Determining whether a sentence is a question or not is decidedly non-trivial. For example, [even the example for this problem in the NLTK book itself](https://datascience.stackexchange.com/questions/26427/how-to-extract-question-s-from-document-with-nltk) only gets a 67% accuracy rate for identifying questions, woefully low for our purposes.

## Why we did what we did

Not even the most evangelical R user would claim that R is a better programming language for natural language processing (NLP) than Python.  I'm sure there are a lot of packages out there that let R interface with Python NLP API, but it's just not the same.  So why did I choose to try this project in R?

Maybe I'm just old-fashioned, but I still put almost all of NLP into the "engineer-y", "machine- earn-y", "let's just get it working" sort of box in my head, as opposed to the "science-y", principled, hypothesis-driven box.[^2]  For Chris and Liz's questions, I wanted to be as "science-y" as possible.  I wanted more certainty about the truth of what we were testing, even if it meant making the problem a little smaller.  I wanted to rely on engineer-y solutions as little as possible.

But also, most importantly, I didn't want to have to *learn* anything.

That's a very un-Zach-like thing to say, but I have to freaking graduate here, guys. I KNOW there's a lot of good out-of-the-box NLP stuff out there, but I didn't want to have to shop around for the perfect library. I didn't WANT to brush up on my Python [NLTK](https://www.nltk.org/) and get that all running.  I wanted this to be EASY.  And principled.  Of course principled.

## Principles and laziness

When I say "principled" I'm referring to that hazy conglomeration of methodological philosophies like starting from well-reasoned hypotheses, using clearly defined and understood assumptions, and not trying to just "make things work", etc.

So after my quick hacks didn't cut it, I decided to _start_ this project as principled and as lazy as possible and work our way down from there.  If we consider all the movie titles with question marks to be questions (a very reasonable assumption, I think), then any identical title *without* a question mark should *also* be a question.  Ta-da, we have our data set ready, with no NLP to be seen and very few assumptions!

Since we had trouble getting financial data, we decided to use average reviews as a measure of success. Whether a movie's average review correlates with financial success is debated and very complicated, but I think most would agree that it is a good measure of at least one type of success.  Although it was relatively arbitrary, we decided *a priori* not to use movie titles with less than five reviews. If fewer than five people reviewed a movie, we just didn't think it would be a reliable estimate.

Unfortunately, after we filtered out movies with less than 5 ratings, there were only 17 titles without question marks. At that point, I just decided to use paired movie titles (differing only by the presence/absence of a question mark), since that type of paired response is essentially the golden standard for these experiments.  Interesting (kind of), principled (mostly), but definitely unsatisfying.  It looked like we were going to have to put in a little effort to make this good.

## Making the problem smaller

Currently, there does not exist any method of determining whether any string of characters constitutes a valid question in English with 100% accuracy. Despite my initial desire not to spend time shopping for out-of-the-box techniques, I did do a little Googling, and pretty much nothing was up to spec.  

But my background is in psycholinguistics and cognitive science. We study what is essentially the [most complex thing in the known universe](https://www.npr.org/2013/06/14/191614360/decoding-the-most-complex-object-in-the-universe). While we will *never* fully understand how the mind works, we can carve out little bits of the question that we can understand a rough approximation of.  So instead of trying to classify *any* type of sentence as being a question or not, I decided that we could use some properties of how some questions are constructed in English to help me classify *some*.  

There are five common interrogative words in English, *who*, *what*, *when*, *where*, *why*, and *how*,[^3] that are used to make "wh-questions".  Unlike yes-no question, wh-questions are asking for specific types of information.  I'm not going to get into the technical details of ["wh-movement"](https://en.wikipedia.org/wiki/Wh-movement) here, but for the most part, when people ask these types of question marks, they begin their sentences with these special words.

But not all sentences that begin with "what" are questions. Case in point, [*What Women Want*](https://www.imdb.com/title/tt0207201/).  But almost *all* sentences that begin with a wh-word immediately followed by a verb[^4] can be interpreted as questions.  This is essentially a grammatical feature of English.[^5] So if we can identify whether the second word is a verb, we can be pretty certain that the sentence can be a question.  Notice, however, that by constraining ourselves to these types of questions, we miss out on movies like [*What Difference Does It Make?*](https://www.imdb.com/title/tt3511442/) and [*How Big Is the Galaxy?*](https://www.imdb.com/title/tt9233172/).  

I was not particularly troubled by this: like I said, I've become used to having to settle for solving the manageable slivers of incredibly difficult problems.  But then Andrew pointed out that pretty much every sentence that starts with "what about..." or "how about..." is *also* a question.

This led to what one might charitably describe as an extended series of passionate debates about design philosophy.[^6]  To make a long story short, I was focused on keeping the false positive rate down while my brother was more concerned with the false negative rate. There's definitely a balance between the two---add too many little "hacks" or patches and the project can become a pile of kludge; make the project to narrow and you might fail to see the big picture.  In the end, we compromised and expanded our definition of "question" to also include some types of inverted yes-no questions as well, using similar (but slightly looser) grammatical constraints.

## Ok, the problem still isn't that easy

Unfortunately, even restricting ourselves to these definitions of "question" didn't make the project a cake-walk. If you recall, I mentioned that the sentences that begin with certain wh-words followed by a non-participle, non-gerund verb are essentially always questions.  To be able to restrict our data to this set, we need to know what part-of-speech (POS) each word likely was. 

To this end, and inspired by a [blog post by Michael Clark](https://m-clark.github.io/text-analysis-with-R/part-of-speech-tagging.html), Andrew came up the Stanford POS tagger from the `openNP` R package.  I stand behind his decision to use this tagger (essentially zero time spent learning how to use it), but this is where the NLP rubber really started hitting the NLP road.

### Problem #1: Tagging was slow as heck

The POS tagger takes a long time to run for a small number of sentences. Not only that, but if you aren't *insanely* diligent about explicit garbage collection, the whole thing crashes R very easily.  Luckily, I have access and experience using my university's remote compute cluster, so after some trial-and-error fiddling with the code (and many esoteric Java errors), I was able to parallelize it over the cluster, reducing the run time from "until the sun dies out" to something like 20 minutes.

### Problem #2: The tagger was not trained on title case

This is probably one of those issues that someone with a keen understanding of `openNLP` and the Stanford parser could easily fix, but it turns out that the Stanford POS tagger we used was sensitive to capitalization.  

For example, we put all our titles into lowercase for standardization when tagging them. But when we did this, the tagger thought i" (lowercase) was a foreign word, and wouldn't tag it properly.  You might think we could just leave the titles with their natural capitalizations, but then the tagger had trouble, imagining that every capitalized word was a proper noun.

Unfortunately, knowing when to capitalize words is *also* a non-trivial problem, since knowing when a word is a proper noun (perhaps a name not found in the dictionary, for example) can be pretty hard. We were mostly able to get around this problem, since we didn't use nouns in our question criteria, although it limited our options substantially.

### Problem #3: Foreign films everywhere

I am (sadly) a monolingual English speaker, and I have mostly forgotten my four years of high school French, two years of Tagalog, and three brilliant months of Esperanto.[^7]  I do not know the ins and outs of how questions are formed in any other language, and our focus was aimed at Hollywood anyway.

The problem is that foreign titles ABOUND in our IMDB data set. Yet IMDB has another data set of alternative, localized titles for movies ([¿Quién engañó a Roger Rabbit?](https://www.imdb.com/title/tt0096438/releaseinfo?ref_=tt_dt_dt#akas)) and which region the title was used in (Mexico). Unfortunately, to the best of our knowledge, it does not appear to have direct information on the language the title was originally written in. For example, [*Das Boot*](https://www.imdb.com/title/tt0082096/) is still referred to as *Das Boot* in English, even though the title is in German.

For a *vast* number of foreign films, our wh-word / yes-no auxiliary verb constraints weeded them out.  To the best of my knowledge, "what" is not a transcription of any word in Hindi, so Hindi titles wouldn't be a problem for the wh-question format.  However, for the yes-no questions, this was not always the case: the "is" in [*Is Raat Ki Subah Nahin*](https://www.imdb.com/title/tt0308417/) does not mean that the title is English.  This was *particularly* annoying for German films---there are quite a number that start with "was" (a cognate of the English word "what"). To make it more confusing, there are also movies that [code switch](https://en.wikipedia.org/wiki/Code-switching) within a single title, like [*Shall We Dansu?*](https://www.imdb.com/title/tt0117615/), or use non-standard English slang, like [*Is Zat So?*](https://www.imdb.com/title/tt0018030/).

## Getting sloppy

I'm going to be honest with you, this is where my principles broke (a little).  First, determine whether a word is English or not, it helps to have a dictionary that includes all word forms (e.g., "run", "runs", "running", "ran" instead of a single entry for "to run"). I was almost about to parse ye olde Wiktionary for all English word forms… but before I got sucked down that rabbit hole, I decided to use a corpus I had worked with previously for a speech perception, the [Carnegie Mellon University Pronouncing Dictionary of American English](http://www.speech.cs.cmu.edu/cgi-bin/cmudict).

A much better and more principled option would have been to use a corpus of English word forms with measure of word frequency as well, such as [SUBTLEX-US, a corpus of movie and TV subtitles](http://www.lexique.org/?page_id=241). That way, we could have avoided the other problems I'm about to discuss.  But the CMUPD was just a tad bit simpler, and we had already narrowed down the possibility space so much, so I just went with that.

When we were testing out my system of removing foreign language titles, Andrew complained that the title [*Indovina chi viene a merenda?*](https://www.imdb.com/title/tt0121402/) wasn't being removed.  When I went to see why, I realized that my system said that the title only had one non-recognized word (within the safe limits): "viene".  I understand that "chi" is the Greek letter (in English), and "a" is also a word, but I was gobsmacked to find that the CMUPD also has entries for "indovina" and "merenda".  Without word frequencies, we couldn't filter out very low-frequency/foreign words.

In the end, the system of foreign word removal I came up with was relatively hacky, very much tailored to the titles we had already narrowed our scope down to, and was especially focused on removing those pesky German titles.  It was a feature-based system that would have been better implemented by something like an XGBoost classifier, but I didn't have the time or the willpower to train something like that for such a small task. It wasn't elegant by any means, but it got the task done.

## Conclusions

So did I end up learning anything?  Sadly, yes.

First, I learned that making a project more principled can actually make it easier sometimes.  Although I could have gone HAM on the question with some "less principled"[^8] NLP methods, there was enough data that I could approach the problem on *my* terms and still get interesting findings.  When I stopped trying hacks and took steps to approach the problem with assumptions I felt comfortable making, things fell into place.

Second, I learned that the inverse is rarely true: the easy route (like using the CMUPD instead of a corpus with word frequencies) can lead to more headaches than it's worth.

Finally, I learned that my brother and I can spend upwards of three hours arguing about what constitutes a "principled" approach and what makes something "hacky".  I mean, this doesn't *surprise* me, but it *was* interesting to go through the process of explicitly negotiating our different design philosophies.  In the end, I think our compromise gave this project the right amount of depth, while still staying manageable.

### *Problem #4: Interactive R plots with Jekyll*

Secret additional problem time!

You might have noticed that some of the plots [in the main post]({{ site.baseurl }}{% post_url 2020-03-27-questionable_movies %}) are interactive.  They use something called ["Plotly"](https://plotly.com/) which is very cool, and easy(ish) to use with `ggplot2`.

_**However**_, these plots are interactive because they are Javascript-powered.  This type of interactive R stuff is super easy to integrate with R Markdown/knitr, but only if you're knitting to HTML.  Unfortunately, my Github Pages site is powered by Jekyll, which means knitting the R Markdown files into Jekyll-flavored markdown.

You'd think this would be an easy problem to solve, but all the (few) example solutions I found online were way too hacky, and involved at some level retooling the entire R Markdown -> blog post workflow.  I eventually cracked how to post these plotly plots the "right" way, and I'll be making a quick post about that later, but dang, it was annoying!

<hr />
<br />

## Source Code:

> [`pos_tagging.R`](https://github.com/burchill/burchill.github.io/blob/master/code/movie_questions/who_let_the_dogs_out_analysis.R)

This is the code I ran on the university compute cluster to do part-of-speech tagging on the movie titles. The output of this code is needed for the next script.

> [`who_let_the_dogs_out_analysis.R`](https://github.com/burchill/burchill.github.io/blob/master/code/movie_questions/who_let_the_dogs_out_analysis.R)

This is the code we used to separate the movie titles that we defined as questions from the others.

> [`2020-03-27-questionable_movies.Rmd`](https://github.com/burchill/burchill.github.io/blob/master/_source/2020-03-27-questionable_movies.Rmd)

If you want to see the source code for any of the cool, interactive plotly graphs or anything else from our "official" post, you can check out the source code above.

### Footnotes:

[^1]: My first hack was to do part of speech tagging by just joining the `tidytext` `parts_of_speech` to the titles---it has a huge list of words and which POS they normally are. Unfortunately, it's very impoverished for what we needed: for example, `tidytext` considers the word "a" to be a noun, verb, preposition, and a definite article (`tidytext::parts_of_speech %>% filter(word=="a")`). Plural nouns were also not included in the database. For example, the word "knights" in [When Knights Were Bold](https://www.imdb.com/title/tt0028495/) comes up as a verb (like "to knight someone") and not a noun, only the singular form of "knight" is interpreted as a person, place, or thing. 

[^2]: Yes, I know there is a lot of cool science at the interface of these two fields, blah blah blah.

[^3]: SCREW *which*, *whom*, *whether*, and *whose*. *Whither* and *whence* are cool though.

[^4]: other than a participle or gerund 

[^5]: *kind of*

[^6]: Peppered with the occasional person insult, of course. We *are* brothers, after all.

[^7]: *Vivu la Esperantuloj!*

[^8]: They're scare quotes, guys. Chill.
