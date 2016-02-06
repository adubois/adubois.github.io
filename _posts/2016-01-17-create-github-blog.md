---
layout: post
title:  "Create a Github Blog"
date:   2016-01-17 20:03:44 +0000
categories: howto
---

Github provides a very efficient worflow for you to host your blog on your own
domain (http://yoursite.yourdomain.com or http://username.github.io).

This howto describes in details how to set up a development environment
on [QubesOS] for you to edit and check your site
before publishing it on [Github]. If you are using a
different OS, most of the points are still useful.

This Howto can be decomposed in the following high level tasks

 * Install required software in Template VM
 * Create App VM to host dev environment
 * Install Jekyll
 * Setup Github pages
 * Setup Git
 * Create Jekyll site
 * Test site
 * Configure site
 * Publish site
 * Setup custom domain

We will install the development environment so that Fedora managed packages are
installed in a template VM (as they are signed) and the Ruby based packages are
installed in a `blog` development VM.

Background
----------

After reading [Joanna Rutkowska]'s [blog post][Joanna new Git based blog], I
decide to do the same and migrate [my Blogger site] to what I suspect is the
same toolchain. As I didn't know any of the tools used, I took the oportunity to
write this post to document the process.

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

[Jekyll] is a system which transforms your plain text( [Markdown] ) into a
static web site and blog. Kramdown (Github's Markdown) is an [easy format] to
use.

Install the Jekyll package from a terminal window in our `blog` VM

```bash
gem install jekyll
```

Setup Github pages
------------------

[GitHub Pages] allows you to host a website directly from your [Github]
repository: "Just edit, push, and your changes are live". We will just add
`sign` between the edit and the push as it is essential that we start to sign
what we produce so that our integrity is respected (assuming that our signing
key can be kept secret... but that is another story).

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
<http://localhost:4000>

Configure site
--------------

We are now going to edit the Jekyll `_config.yml` file to start personalize you
site

```yml
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

```bash
gs
ga .
gc -m "Site configuration initialized"
```

Publish content
---------------

Now that you have tested and commited your changes we can publish them (git
push)

```bash
gp master
```

Setup custom domain
-------------------

Instead of hosting your site at https://username.github.io, you can setup a
custom domain such as http://yoursite.yourdomain.com.

Note that this will remove the `s` in https but blogs integrity should be
verified using Git signed commits rather than relying on a broken public
certificates infrastructure. However Github may allow in the future the upload
of private key and certificate to put back some level of security.

Custom domain can be setup by adding a `CNAME` file (DNS alias)

 * to your Github repository and
 * to your DNS zone at your DNS provider

The former to allow Github

 * to redirect to yoursite.yourdomain.com users who went to username.github.io
 * to serve your site as a load balanced virtual domain

The second to

* direct users to Github's server IPs

Add a `CNAME` file containing

```
yoursite.yourdomain.com
```

Publish this change

```
ga .
gc -m "Adding custom domain"
gp
```

Finally, you will have to configure the CNAME with your DNS provider and you may
[find these tips] useful.

Looking forward  to read your future posts ;-)

[QubesOS]: https://www.qubes-os.org/
[Joanna Rutkowska]: http://blog.invisiblethings.org/about/
[Joanna new Git based blog]: http://blog.invisiblethings.org/2015/02/09/my-new-git-based-blog.html
[my Blogger site]: http://bowabos.blogspot.co.uk/
[Github]: https://github.com
[Qubes's getting started]: https://www.qubes-os.org/getting-started/
[Jekyll]: http://jekyllrb.com
[Markdown]: https://daringfireball.net/projects/markdown/
[easy format]: http://kramdown.gettalong.org/syntax.html
[GitHub Pages]: https://pages.github.com
[Git]: https://en.wikipedia.org/wiki/Git_%28software%29
[find these tips]: https://help.github.com/articles/tips-for-configuring-a-cname-record-with-your-dns-provider/

[start with Git]: /howto/2016/01/16/starting-with-git/
