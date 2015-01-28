---
layout: post_page
title: Running a blog with Jekyll on Windows without Jekyll
---

I've been meaning to set up a blog for a while, but have just been put off by the big systems because they just feel a little
too large, and I don't want to be reliant on some CMS company having their websites running. I always thought static generation
is a good idea, and when I heard about Jekyll I thought it sounded perfect for what I wanted, which was to be able to store
my posts in a Git repo as simple markdown and HTML.

However, I'm on Windows, and don't want to set up Ruby etc just to have a blog, so this is an experiment in running a Jekyll
site on Github without actually running Jekyll locally. This may well turn out to be a bad idea, but I'm giving it a try anyway.
For those not in the know, when you upload to a special Git repo in GitHub, it gets treated as a Jekyll site. Github will run
the transformation from templates and markdown into a bunch of static HTML and publish it at quite a nice URL automatically. 

Step one: create a repo called `<your github username>.github.io`. Here is my one:
[https://github.com/danielflower/danielflower.github.io](https://github.com/danielflower/danielflower.github.io)

Step two was getting bootstrapped. I decided to just grab a simple design from [jekyllthemes.org](http://jekyllthemes.org/) and went
with [richbray.me/frap/](http://richbray.me/frap/) in the end. I just downloaded the theme and stuck it in the Git repo
I just cloned. I pushed and cloned, then waited for a few minutes, and then I had a blog at `http://danielflower.github.io/`. Cool!

Then I just made a few changes, pushed again, and this blog exists.