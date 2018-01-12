---
layout:  post
title: "How to Share a Blog with an Asshole: #BurchillBoizBringDaNoise"
comments:  true
published:  true
author: "Zach Burchill"
date: 2017-08-18 10:00:00
permalink: /blogshare/
categories: [git,'github pages','#burchillboiz','#bringdanoise',Zach,Andrew,Burchill]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---

{% assign data_file = site.pages | where: "type", "burchilldata" | first %}
{% if data_file and data_file.authorlinks['Andrew Burchill'] %}
{% assign andrew_url = data_file.authorlinks['Andrew Burchill'] %}
{% endif %}

Everyone, meet [Andrew Burchill]({{ andrew_url }}), my (actual) twin brother. My baby brother (by sixteen minutes). "Lil' Andrew," we used to call him. If you want to learn more about him, just jump over to [his site]({{ andrew_url }}) and check him out. 

If you do, you think that his site looks a lot like mine. I wonder whether he forked his site from anyone...? _Hmmm..._   Anyway, because our websites are so compatible, we embarked on a journey to see if we could somehow share a collaborative blog while maintaining our separate fiefdoms.

_**This**_... is the story of that journey.

<!--more-->




