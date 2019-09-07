---
layout:  post
title: "Raspberry Pi 4 Bluetooth Scale"
comments:  true
published:  true
author: "Zach Burchill"
date: 2019-09-07 10:00:00
permalink: /bluetooth_scale_intro/
categories: ["raspberry pi",hardware,python,scale,health,DIY,bluetooth,hack,"wii fit",wii]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---



I'm getting fat.  Well, fatt<em>er</em>. My [twin brother and perfect control experiment](https://andburch.github.io/about/) recently lost a lot of weight dieting, which got me thinking about going on one myself. As a data scientist / machine learning engineer, I saw this as a perfect opportunity to get some good data.

I've spent three months working on a Bluetooth weight collection system and zero months on a diet, so without further ado, let me walk you through starting up your very own Bluetooth bathroom scale. 

<!--more-->

First, in order to collect and store weight information, you need something that will collect the information and something that will store the information (lol). Let's focus first on the data collection component.

## Bluetooth scales

Why Bluetooth? If you want to store weight data, you have to _get_ it, and unless you wire your scale to a data storage system, Bluetooth is the way to go. You can save yourself a lot of time (and honestly, money) if you just buy yourself a bathroom scale with Bluetooth capability and that can record your weight. But I wanted to really get _into_ that weight data, and I couldn't find one that let me have the end-to-end control that I wanted.

While there are some Bluetooth scales that have been hacked to be open source,[^1] I couldn't find any of them being sold currently online. What I did find were many articles about hacking [Wii Fit Balance Boards](https://en.wikipedia.org/wiki/Wii_Balance_Board) into Bluetooth scales.

### Wii Fit Boards

The Nintendo Wii syncs its controllers to the console via Bluetooth, and the Wii Fit board is a controller that is basically just a scale. The Wii controllers were so interesting (cheap, mass-produced motion/gesture devices) that hackers quickly reverse-engineered how they worked and have [compiled super detailed information how to use them with homebrewed setups](https://wiibrew.org/wiki/Main_Page).

Let me tell you, there are a _lot_ of tutorials out there for turning a Wii Fit boards into scales. Here's a list of some of what I found:

 * Greg Nichols' [Initial State weight tracker on ZDNet](https://www.zdnet.com/article/diy-build-a-hackable-weight-tracking-scale-with-a-sense-of-humor-using-raspberry-pi/) (based on Stavros' work):
 * Stavros Korokithakis' [internet-enabled bathroom scale](https://www.stavros.io/posts/your-weight-online/)
 * ReducedGravity's [game controller](http://reducedgravity.net/wiipi/) (also based on Stavros' work)
 * Matt Cutts' [Linux tutorial](https://www.mattcutts.com/blog/linux-wii-balanceboard/)
 * Marcel Bieberbach's [`wiiweigh`](https://github.com/chaosbiber/wiiweigh)
 * Marcel Lauhoff's [`wiiscale`](https://github.com/irq0/wiiscale)

Great, right?  (~~_Heh heh, NO, you cretin, you worm, your pain is just beginning._~~)  I bought one for $22 off eBay. So let's talk about the data storage system.

## Raspberry Pi

We need something to communicate with the Wii Fit Board and store / process the weight data, and the Raspberry Pi is perfect for that. The [Raspberry Pi](https://www.raspberrypi.org/) is a line of very cheap, very small computer boards that are designed for people who want to learn how to code and do cool hardware projects like this. I've always wanted to get one, they're perfectly suited for this project, and the new model (the Raspberry Pi 4) just came out and everyone was raving about it. So I bought one.[^2]

So I had my data collection and storage systems, a bunch of great tutorials, and I was ready to go! Right? Sadly, no.

## Problems

### Bluetooth support for Python sucks

Python is great---it's used so often for so many applications, it has robust libraries and tutorials for just about everything! Sadly, "everything" doesn't include Bluetooth. If you want to work with Bluetooth, learn C.

This is summing up a _huge_ amount of digging I had to do, so pardon my inaccuracies, but among the many half-lived, stunted Python Bluetooth libraries, [PyBluez](https://github.com/pybluez/pybluez) is king. It's built on the [Bluez](http://www.bluez.org/) Bluetooth stack for Linux, and is by far the most-used Python Bluetooth library.

It's also no longer under active development and basically only works for Python 2.7. It's pretty out of date, and almost every relevant Python Bluetooth tutorial references required code that no longer exists or is seriously broken. I wanted to make a shiny, Python 3 module that would work with Bluetooth and I was utterly defeated. 

I tried for so long, reverse engineering C code into Python, trying different packages, cobbling different tutorials together, and nothing worked. As we shall see later, this was a waste on _multiple_ levels.

### Basically only one tutorial really works

Remember those tutorials I listed earlier? They're all outdated or janky as hell. When you turn on the Wii Fit Board, unless it's "paired" to a device you need to press a red "sync" button in the battery case to make it discoverable so you can connect to it. Every time. What we want is a system that you can turn on and connect automatically.

Greg Nichols' way around this is to _tape a pencil to the bottom of the board_ to press the button (grotty), Stavros Korokithakis straight-up admits he can't solve this issue, and most of the other tutorials just don't work anymore anyway.

The only one that works with the Raspberry Pi 4 with Bluez 5 (the most recent version) is Marcel Bieberbach's `wiiweigh` (the last one I found). He includes [a step-by-step guide](https://github.com/chaosbiber/wiiweigh#howto) to install all the necessary components---and more importantly---to _pair_ the Wii Fit Board with the Raspberry Pi.

## Solution

This is the holy grail of Wii Fit projects: once paired, turning the Balance Board on makes it automatically connect to the Pi, and disconnecting it (via the Pi) automatically turns it off. To get his tutorial to work, I had to change a few steps, however.

For some reason, `xwiimote` and the `xwiimote-bindings` weren't building properly, so following the directions from [this stackoverflow question](https://askubuntu.com/questions/633949/failed-to-build-xwiimote-bindings), I changed the `./autogen.sh` lines for both to `./autogen.sh --prefix=/usr` and it worked!

I'll be writing more about what I've done with this setup---what I've talked about here is only the process to get things _started_. It's looking very cool so far, so it should be good.

### Post-script: Bluetooth isn't the issue

After I got Marcel's code working, I still wanted to write my own Python 3 version of his code and simplify it. I'll keep things short here, but it turns out worrying about PyBluez was a red herring. Once you pair the Wii Fit Board with the Pi, it essentially acts as an "HCI" device and behaves totally differently. Stavros actually mentions this in his blog post, but I didn't understand at the time.

After incalculable pain[^3], I realized that the package I should be focusing on is Python's `evdev` library ("event devices", what are those?). At this point, I still don't understand 90% of what it is, but I was actually able to communicate with the board. For right now, I'm using a modified version of Marcel's Python 2 package to communicate with the board and Python 3 to handle the storage and processing (more on that in the future), but hopefully I'll eventually be able to get it all in Python 3.


<hr />
<br />

## Source Code:

The source code for getting the basics set up is [here](https://github.com/burchill/wiiweigh), but it's almost exactly what Marcel did, but tweaked ever so slightly. I'm making some _siiiick_ Flask stuff to interface with the scale, but I'm not ready to share that with people.

### Footnotes:

[^1]: For example, those supported by the [openScale](https://github.com/oliexdev/openScale) project.

[^2]: My advice is: go with a kit. I had to order the power supply, SD card, and HDMI connectors all separately, and it took _forever_ to come in.

[^3]: Yeah I'm a drama queen, but this had me at my wit's end. I was going through the source-code for random C packages looking for things I didn't understand, stumbling through weird Unix stuff, smashing my head against the wall with architectures I didn't understand. 
