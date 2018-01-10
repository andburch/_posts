
 # Zach and Andrew's blog collab
 
 ## What we *absolutely* need to do:
 
 ### Making the submodules work
 
 At this point, we pretty much *have* to make both `_post`s and `figures` git submodules.  
 
 **Problem**, this means we can't have blog posts use the images in `images`.
 
 **Solution?**: we move the static images used for the blog posts into a subfolder in `figures` called `figures/static_images`. Pretty sure the `knitr` stuff won't delete anything in that subfolder and that putting a subfolder there won't hurt anything, but who knows. Reading a bit more about Jekyll stuff, I'm actually not so sure.
 
 ### Adding double burchill nav bar
 
 We need to get a navbar for the blog posts that scales with a shrinking window width
 
 ## What we definitely *should* do:
 
 ### Come up with a good way of sharing necessary code
 
 Making anything look good and be robust is going to make sharing code necessary.
 
 Here are things we'd want to share and why we'd want to:
 
 * **Basic CSS code/variables (color schemes, etc.)**: If we want to have the individual posts we make have different color schemes that match our websites, it would be dumb not to have these be in a shared repo. If you ever change your color scheme, etc., it would look super dumb not to have it automatically update the blog posts.  
 * **Links in the shared navbar**: Honestly, we almost absolutely need these to be shared. Preferably as liquid variables.
 * **Jekyll/liquid layouts**: if we want to implement color schemes for different authors, we need to share the markdown layouts that do it
 
 ### Make all the posts use different layouts, as determined by author
 
 We need to do something like this on all the posts (or use a sneakier way) that we make so that Jekyll knows to use different color schemes. At most this will just be changing something in the YAML front matter for each post.
 However, I think getting a good working version of this might actually be trickier than we thought.
 
 # Helpful links
 
 * Read through this first: https://jekyllrb.com/docs/configuration/
 * Variables in Jekyll/liquid: https://jekyllrb.com/docs/variables/
 * Custom data in liquid: https://jekyllrb.com/docs/datafiles/
 
 # Action items:
 
 We should divy up what needs to be done.  Here are some preliminary tasks that I've broken down. When you're done with the task, check the box.
 
- [X] Test checkboxes: **Zach**
- [ ] Make the double navbar: **Andrew**
- [ ]  See if you can make a post use a different color-scheme/layout based on the author (see https://jekyllrb.com/docs/configuration/): **Andrew**
- [X] Test if making a new `.md` post from a `.Rmd` file will erase static images in the `figures` folder: **Zach**
     - It doesn't.
- [ ] Get some way to share Liquid data: **Zach**
- [ ] Come up with a system for managing the CSS/layout differences across sites: **Andrew**
     - ~~For this, it'll be pretty easy. `general.scss` is the only file that uses the CSS variables except one reference to `$text-color` in the `nav.css`. So if we just need to have a `generalA.scss` and a `generalZ.scss` and have an `if` statement in Liquid, based on the front matter of the blog post.~~ 
     - I don't think that's a good idea, since we'd then have to make changes to both files whenever we wanted to change something. So far, the best solution I can see is to make `general.scss` a partial file (`_general.scss`) and have what is *essentially*--but not *actually* two partial files load both. So a `zstyle.scss` file that basically only `@imports` `_variables.scss` and then `_general.scss`, and a `astyle.scss` file that does the same thing, but then `@imports` another, shared partial file instead (hypothetically, `_andrew_variables.scss`, which exists in a shared folder) that redefines the variables as the other twins' style things.
        - So what you're saying is the `[a/z]style.scss` is just a wrapper that adds the variables at the very end? 
        - [X] **To test:** Can a partial file `@import` another partial file?  **A partial file CAN load another partial file, seemingly**.
        - [X] ~~**To test:** I believe importing a file which defines the variables differently *should* overwrite variables from previous `@imports`, but I'm not sure.~~  This **does NOT work**.
            - [ ] Yeah, but if you include a stylesheet in an HTML document, then include another stylesheet in the next line that conflicts with the first, the second one takes precedence, I believe. So, no need to overwrite "variables", just totally overwrite the CSS via the order of the `<link rel="stylesheet" ...` tags. Liquid couldn't do the other thing anyway. 
        - ~~Also, organizationally, it would be best to move all versions of the `_variables.scss` file to the shared repo, making them something like `_andrew_variables.scss` and `_zach_variables.scss`.~~
       ### Note: are you going through all this work JUST to have THREE (3) CSS variables (that don't change often) update easier?

- [X] Test if you can put `figure` in `_posts`: **Zach**
     - It doesn't work. At least, I don't think it does. Jekyll doesn't like it.
- [X] Find a way of putting `figure` in `_posts`. (Try this: https://nhoizey.github.io/jekyll-postfiles/): **Zach**   
    - But how do you get the figures INTO those files? Like, when you knit the .Rmd files, how do they know where to go? 
- [ ] Standardize naming conventions and image path stuff: **Zach**
     

 
 
