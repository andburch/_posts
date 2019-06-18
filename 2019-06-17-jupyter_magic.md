---
layout:  post
title: "Making 'Magic' with Jupyter Notebooks"
comments:  true
published:  true
author: "Zach Burchill"
date: 2019-06-17 10:00:00
permalink: /jupyter_magic/
categories: [R,python,"jupyter",magic,"jupyter notebook","R Markdown",presentation,reports,tutorial,"machine learning"]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---



At my new machine-learning job (internship), I use a lot of Jupyter notebooks.  If you don't know what a Jupyter notebook is, it's kind of like a more interactive version of an R Markdown sheet, but for Python. They're great, but there were a few features (or lack thereof) that really got on my nerves. Luckily the stuff under the hood of Jupyter notebook is crazy flexible, and with a little know-how we can jerry-rig us some cool stuff.

Specifically, we can use IPython's "magic" commands.

<!--more-->

## Magical mystery tour

Jupyter notebooks run on a form of Python called IPython, and IPython has something called "magic commands," which do weird cool things. My only experience with these before was using `%paste`, which you can use to paste multi-line blocks of code from the clipboard into a terminal.

What's great is that [you can use these commands in Jupyter](https://towardsdatascience.com/the-top-5-magic-commands-for-jupyter-notebooks-2bf0c5ae4bb8). For example, you can time how long a cell (the Jupyter version of a code chunk) takes to run by putting `%%time` at the top of the cell above the code. 
You can even run arbitrary _R_ code in Jupyter cells [via `rpy2`](https://towardsdatascience.com/the-top-5-magic-commands-for-jupyter-notebooks-2bf0c5ae4bb8), by putting `%%R` at the top of the cell, and can pass variables back and forth between the R and Python kernels. It's seriously crazy and quite "magical."

But you can also make your _own **custom**_ magic, which is where things can get full-on _HARRY POTTER_.

## Custom magics

My first problem: I was running a bunch of complicated models on large amounts of data, and they would take a while to complete. With R, I've written my own beeping functions that will alert me auditorily when big jobs have finished. 

But 1) my Jupyter notebooks were running on remote servers, so making the machine beep wouldn't help, and 2) I wanted things to stay clean and organized in the notebook.  If I could _time_ how long a cell takes to run with magic, why couldn't I have it _alert_ me when it finished?

There are some pretty good introductions to making your own custom magics for Jupyter, for example one by [Keita Kurita](https://towardsdatascience.com/the-top-5-magic-commands-for-jupyter-notebooks-2bf0c5ae4bb8) and a more advanced one by [Cyrille Rossant](https://ipython-books.github.io/14-creating-an-ipython-extension-with-custom-magic-commands/), but they're a little short. I had to do a little hacking to get something that I wanted, digging through the old source code of some of the old IPython magics.

## Play a beep after execution

Basing my general structure off Cyrille's work, I came up with the following skeleton:


```python
from IPython.core.magic import Magics, magics_class, line_magic, cell_magic
from IPython.utils.capture import capture_output

@magics_class
class MyMagics(Magics):

    @line_cell_magic
    def beep(self, line, cell=None):
        exec_val = line if cell is None else cell
        with capture_output(True, False, True) as io:
            self.shell.run_cell(exec_val)
        # Make a beep here somehow ?
        io.show()
```

To get it to beep, I realized I could call `IPython.display.Audio("beep.wav", autoplay=True)`, which would render a big HTML `<audio>` element and play it immediately after the code was done.[^1] But I wanted a _pristine_-looking notebook, not one cluttered with big ugly audio players

I did something that forms the background of the hacks in this tutorial---I started messing with the HTML.

Since you can have arbitrary HTML in the output, and since with the newest versions of HTML you can put CSS styles in the body of the page, I just had the `beep` function set the CSS for `audio` elements to `display: none`, hiding them from view.  

_Et voilà:_

```python
# ...
@line_cell_magic
def beep(self, line, cell=None):
    exec_val = line if cell is None else cell
    
    with capture_output(True, False, True) as io:
        self.shell.run_cell(exec_val)
    self.shell.run_cell('from IPython.display import Audio; Audio("beep.wav", autoplay=True)')
    display(HTML('<style> audio { display: none; } </style>'))
    io.show()
```

I also thought it would be neat to be able to play a beep _and_ time the code, so I cooked up a lazy hack that literally just gets a cell and adds the two magic commands to it:

```python
# ...
@line_cell_magic
def time_beep(self, line, cell=None):
    exec_val = line if cell is None else cell
    self.shell.run_cell("%%time\n%%beep\n{}".format(exec_val))
```

After saving the code as `zachmagic.py`, I could make my custom magics be available throughout a notebook by running `%load_ext zachmagic` in the first cell.

For example, a cell with the following would alert me after the model is finished fitting, and tell me how long it took:

```python
%%time_beep
m = pygam.LinearGam()
m.fit(X, y)
```


But there are even crazier, hackier possibilities for modifying Jupyter notebooks!

## Making Jupyter notebooks look "publishable"

Another annoyance I encountered with Jupyter notebooks was that I couldn't knit them into "final products" like I could with R Markdown files (e.g., into PDFs).[^2]  My boss wanted me to put the results of a report I had made into a Doc file instead of sending him a link to the notebook, and it was only then that I realized how _ugly_ a sufficiently long Jupyter notebook can be--the input cells take up so much space with no easy way to hide them.

I wanted an ability analogous to `knitr`'s `opts_chunk$set(echo = FALSE)`, where I could hide all the inputs with a single command. For example, when I'm done editing a notebook and want to share it with other people.

Thinking back to how I hid the `<audio>` element, I decided to do something similar, but for the HTML that made up the input cells:

```python
# ...
@line_magic
def hide_all(self, line, cell=None):
    display(HTML('''<style> div.input { display: none; }</style>'''))
    
@line_magic
def show_all(self, line, cell=None):
    display(HTML('''<style> div.input { display: flex; }</style>'''))

```

Unfortunately, running `%hide_all` in a cell hid all the inputs, _including_ its own, which made it really hard to turn off. While this is not a perfect solution, I ended up adding a print statement that would output "Jupyter inputs set to hide via this cell", so that you could click on the cell and then delete it to restore the inputs.

## Even more complicated!

I wanted to better capture `knitr`'s chunk option functionality though. Specifically, I wanted to control _which_ inputs were shown and which were hidden, like you can do with the `echo = TRUE` chunk option, instead of blanket-hiding them all.

In order to do _that_, I made a magic command that would display the _input_ code of the cell in the _output_ automatically, but would _hide_ it unless all the other inputs were hidden. This way, the cell's input would still be visible in the final product no matter what. I used IPython's built-in code displayer, but edited its output to add in a custom class name that would toggle with `%hide_all`.

Unfortunately, this had very minor visual changes to other output elements in the rest of the document (i.e., the background of the tables slightly darkened). Being too OCD to let that slide, I further hacked the output so that it would generate a unique class name for each output cell and only apply the stylings to that particular cell/class.

Check out my final code in the source below, and you can get started yourself!


<hr />
<br />

## Source Code:

The source code is on [my GitHub as a gist here](https://gist.github.com/burchill/4adb9531a246e8c27752c1b19e0236bb) with relatively extensive comments. Feel free to use / modify it for your own purposes!

### Footnotes:

[^1]: Of course, you have to save an audio file called `beep.wav` wherever you're saving your Jupyter notebooks for this to work.

[^2]: For the record, I think you can do something like this with `nbconvert`, but meh... 
