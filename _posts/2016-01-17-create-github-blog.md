---
layout: post
title:  "Create a Github Blog"
date:   2016-01-17 20:03:44 +0000
categories: Howto
---

Github provide a very efficient worflow for you to host your blog on your own
domain (http://yoursite.yourdomain.com or http://username.github.io).

This howto describes in details how to set up a development environment
on [QubesOS] for you to edit and check your site before publishing it on Github.
If you are using a different OS, most of the points are still useful.

This Howto can be decomposed in the following high level tasks

 * Install the required software in a Template VM
 * Create a VM to host the development environment
 * Install Jekyll
 * Setup Github pages
 * Setup Git
 * Create Jekyll site
 * Test site
 * Configure site
 * Publish site

We will install the development environment so that Fedora managed packages are
installed in a template VM (as they are signed) and the Ruby based packages are
installed in a `blog` development VM.

Install the software
--------------------

Install the Ruby development package from a terminal window in your template VM

```bash
sudo yum install ruby-devel git
```

Create a blog VM
----------------

Create a new `Blue` (showing that it is a pretty safe environment as we'll be
the one generating files and the only interraction after set-up should be with
our Github repository), `blog` VM as documented in [Qubes's getting started]
page using the default Fedora template.

Install Jekyll
-------------

[Jekyll] is a system which transforms your plain text ( [Markdown] ) into
a static web site and blog. Markdown is an [easy format][Markdown Cheatsheet]
to use.

Install the Jekyll package from a terminal window in our `blog` VM

```bash
gem install jekyll
```

Setup Github pages
------------------

[GitHub Pages] allows you to host a website directly from your [Github]
repository: "Just edit, push, and your changes are live". We wll just add sign
between the edit and the push as it is essential that everybody start to sign
what they produce so that our integrity and privacy is respected.

Assuming your have a `username` Github account, please create via the Github web
site a new repository named `username.github.io`
 * Which is Public
 * With a .gitignore for Jekyll

Setup Git
---------

[Git] allows you to manage source files and sign your commits.

Let's first follow the instructions in this previous Howto post:
[start with Git], to setup your Git environment.

Create Jekyll site
------------------

Let's first clone the repository we created on Github locally from a terminal
window in the 'blog' VM

```bash
git clone https://github.com/username/username.github.io
```

Let's now initialize the repository with a new Jekyll site

```bash
cd username.github.io
jekyll new . --force
```

Let's check the status of our local Git repository, stage the files (git add),
and commit the changes to our local repository

```bash
gs
ga .
gc -m "Initial Jekyll site commit"
```

Test site
---------

To preview our changes before publication we can use our local Jekyll
installation to generate the site and browse it locally.

Open a new terminal window in your `blog` VM

```bash
cd username.gihub.io
jekyll serve
```

Jekyll now serves your local development site locally (you just need to press
Ctrl+C to stop it). You can browse to it from firefox in the `blog` VM via
[http://localhost:4000]

Configure site
--------------

We are now going to edit the Jekyll `_config.yml` file to start personalize you
site

```
# Welcome to Jekyll!
#
# This config file is meant for settings that affect your whole blog, values
# which you are expected to set up once and rarely need to edit after that.
# For technical reasons, this file is *NOT* reloaded automatically when you use
# 'jekyll serve'. If you change this file, please restart the server process.

# Site settings
title: My blog site
email: myEmail@mydomain.com
description: > # this means to ignore newlines until "baseurl:"
  Blog covering QubesOS, personnal IT security and ho to provide services from your server to your home devices, with a very strong focus on security.
baseurl: "" # the subpath of your site, e.g. /blog
url: "http://username.github.io" # the base hostname & protocol for your site
twitter_username: yourTweeterAccount
github_username:  username

# Build settings
markdown: kramdown
```

Jekyll usually picks up automatically changes made to local files with the
exception of this `_config.yml` file. Press Ctrl+C to stop the deamon are
restart it (this will not be required for different files updates).

Refresh your browser and "Voila".

Once you are happy with the changes, stage and commit them
```
gs
ga .
gc -m "Site configuration initialized"
```

Publish content
---------------

Now that you have tested and commited your changes we can publish them (git
push)

```
gp master
```


[QubesOS]: https://www.qubes-os.org/
[Qubes's getting started]: 
[Jekyll]: http://jekyllrb.com
[Markdown]: https://daringfireball.net/projects/markdown/
[Markdown Cheatsheet]:
https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet
[GitHub Pages]: https://pages.github.com
[Github]: https://github.com
[start with Git]: /howto/2016/01/16/start-with-git/