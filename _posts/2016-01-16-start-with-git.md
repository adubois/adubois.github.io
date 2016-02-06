---
layout: post
title:  "Start with Git"
date:   2016-01-16 7:03:44 +0000
categories: howto
---

[Git] is a distributed versioning system. It started as a tool developed to
maintain the Linux kernel. It has been found suitable to manage documentation,
blog posts, static web sites and even microkernel binaries.

If you want to use Git, as I [now do], here are the steps required to get
started

 * Install
 * Configure
 * Create Github repository
 * Create local clone
 
This post is based on:

 * [Git Howto]
 * [Github Bootcamp]


Install
-------

To install Git as a QubesOS users, from a terminal window in the template VM

```bash
sudo yum install git
```

Configure
---------

In QubesOS, the configuration should be done in an App VM as these settings are
user specific (stored in the home folder linked to /rw/home) and will therefore
survive a reboot. Moreover these settings should not be visible in all your VMs,
particularly the risker ones.

Type the following from a terminal window in an App VM to idetify yourself,
cache your credentials for one hour, and terminate line correctly:

```bash
git config --global user.name "First Last"
git config --global user.email "user@domain.com"
git config --global credential.helper 'cache --timeout=3600'
git config --global core.autocrlf input
git config --global core.safecrlf true
```


Note that if the `git` command cannot be found, you may need to
restart your App VM if you installed it in the template VM with the App VM
running.

To make our life easier, let's set up few aliases:

```bash
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.br branch
git config --global alias.hist 'log --pretty=format:"%h %ad | %s%d [%an]" --graph --date=short'
git config --global alias.type 'cat-file -t'
git config --global alias.dump 'cat-file -p'
```

Let's add some even shorter aliases in our shell by editing `.bashrc` in
our home folder, please note that this may hide command you are using (i.e. gs
for ghostscript):

```bash
alias gs='git status '
alias ga='git add '
alias gb='git branch '
alias gc='git commit'
alias gd='git diff'
alias go='git checkout '
alias gp='git push origin '
alias gh='git hist'
```

Create Github repository
------------------------

If you want your content to be hosted in a public repository, Github is one of
the natural choice at the moment. A public repository may be a good choice so
that others can fork your repo and either:

 * Contribute to it by submitting `pull requests` or
 * Fork it to start a new project and leverage on your work

Follow the instructions in the [Create a repo] page from Github to:

 * Create an account
 * Create a repository `myrepo`

Create local clone
------------------

From a terminal window in an App VM

```bash
git clone https://github.com/username/myrepo
```

This is it to get started with Git.

[Git]: https://en.wikipedia.org/wiki/Git_%28software%29
[now do]: https://adubois.github.io/howto/2016/01/17/create-github-blog
[Git Howto]: http://githowto.com/
[Github Bootcamp]: https://help.github.com/categories/bootcamp/
[Create a repo]: https://help.github.com/articles/create-a-repo/

