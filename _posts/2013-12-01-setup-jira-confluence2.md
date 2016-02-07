---
layout: post
title:  "How-to set up Jira and Confluence on Fedora VM in Qubes OS"
date:   2013-12-01 21:03:44 +0000
categories: howto
---

This article describes how to set up  [Jira] 6.1.4 and [Confluence] 5.3.4
(hosted on [Tomcat] 7.0.42 (running on Java OpenJDK JRE 1.7.0_45) with
[PostgreSQL] 9.2.5 as a database back-end and [apache httpd] 2.4.6 as a reversy
proxy front-end with SSL) on [QubesOS] v2 Beta 2 (Fedora 18)

Audience: Unix/Web admin who are trying to set these services up on Fedora,
Fedora on Qubes or who want to get familiar with how to set-up a service on a
QubesOS VM.

Implementation time: If you are an old developer and know the value of
copie/paste to avoid problems, it should take you between 1 and 2 hours.

Jira and Confluence provide similar functionality to [trac] (bugs/issues
tracking and wiki). It is free for open source projects. You however have to pay
for a licence otherwise. There is a 30 days trial licence if you want to check
it out before you buy. I am using the 2x $10 starter licence for 10 users and
find such investment very valuable. Moreover for this licence, all benefits are
going to the Room to Read charity, which promotes literacy and gender equality
in the developing world.

[QubesOS] is at the time of writing, to my view, one of the most secure Linux
based desktop operating system. It can run Linux and Windows guest virtual
machines and isolate VMs, networking, usb and graphical stacks levering on Xen
hypervisor. It also leverage on a templating system to minimize the disk
footprint of VMs based on the same template as well as patcing. It does not
support 3D graphical operations out of the box but advanced users can set-up a
second Video card and dedicate it to one of their VM (and one monitor or another
port in your monitor). It is aimed to be used by IT security specialist on the
go but it is also a very nice fit as a home desktop that you can leverage on to
host services (file share, web site, etc...) more securely while browsing the
dark net or the red district at the same time on the same hardware.

This blog post is discussed in the Qubes user group [here][Qubes user group].

Target architecture
-------------------

### Risk analysis ###

The only information of value is the data in the database.

If Apache is compromised, The attacker would be able to impersonate any active
user, and at some point be able to retrieve all the data in the database.

If Jira or Confluence is compromised, the credentials to the database would
therefore be compromised as they have to be readable by the application daemon.
Even if encrypted with a pass-phrase provided during boot time, a valid session
is present in memory.

We are therefore for now going to run Apache, Jira, Confluence and PostgreSQL
in the same VM.

Note: Based on the information provided in this tutorial it would however be
easy to split into a three tier model Apache, Jira/Confluence, PostgreSQL with
two legs (Jira/PostgreSQL and Confluence/PostgreSQL) in a Y shape.

### Methodology ###

QubesOS offers template VM which present its file system to other VM in read
only mode and allow patching of fedora packages. We will therefore leverage as
much as possible on this by using fedora packages whenever possible.

In order to build and be able to test the various components as we stack them
up, we will start with the far end by setting up the database, then Jira and
Confluence and finally the reverse proxy which will sit in front.

Preparing the Template
----------------------

### Installing PostgreSQL ###

Launch the terminal in the template.

 * Click Application Menu (top left)
 * Select Template fedora-18-x64 / fedora-18-x64: Terminal

Install the PostgreSQL Server by typing in the template Terminal:

'''bash
sudo yum install postgresql-server
'''

### Installing Tomcat ###

Atlassian states that only Oracle JRE is supported. I am going to take the risk
of using OpenJDK instead as I will have automatic update via fedora patches. I
will also use the Tomcat package provided by fedora for the same reasons.

Note: If you want to use the Oracle JRE, do not install it in the template but
in the app VM unless your are ready to patch it when required to ensure all your
VMs use a patched version or if you are building a dedicated template.

Let's install tomcat

'''bash
sudo yum install tomcat
'''

In my case this installs Tomcat 7.0.42 and will use OpenJDK Runtime Environment
1.7.0_45.

Let's create the Jira and Confluence user

'''bash
sudo /usr/sbin/useradd -r --comment "Account to run JIRA" --shell /bin/bash jira
sudo /usr/sbin/useradd -r --comment "Account to run Confluence" --shell /bin/bash confluence
'''

### Installing Apache httpd ###

Let's install httpd and make sure we can SSL enable it

'''bash
sudo yum install httpd mod_ssl openssl
'''

In my case this installs httpd 2.4.6


For these changes to be visible to the wiki VM, shutdown the template VM.

'''bash
sudo halt
'''


[Jira]: https://www.atlassian.com/software/jira
[Confluence]: https://www.atlassian.com/software/confluence
[Tomcat]: http://tomcat.apache.org/
[PostgreSQL]: http://www.postgresql.org/
[apache httpd]: http://httpd.apache.org/
[QubesOS]: http://qubes-os.org/
[trac]: http://trac.edgewall.org/wiki/TracInstall
[Qubes user group]: https://groups.google.com/forum/#!topic/qubes-users/2gHjwj3YRrI

