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

Install the PostgreSQL Server by typing in the template's Terminal:

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

Preparing the Wiki VM
---------------------

### Creating a Wiki VM ###

In Qubes OS, this is trivial. In Qubes VM Manager:

 * Select in the menu VM / Create AppVM
 * Give it the name wiki
 * Select the color blue
 * Launch the creation by pressing OK

Note: The attack surface of this VM is of medium size. However the threat is
fairly small as I am planning on using this infrastructure only for personal
use and will either access it within the same QubesOS with a Disposable VM or
possibly remotely after having connected to it via a reverse proxy hosted as a
VM in my Qubes server with Client IP and Certificate validation and a
Disposable VM Browser as a client. I have therefore used the blue color.

Preparing the PostgreSQL databases
----------------------------------

### Preparing the read-write partition ###

In the read-write partition, let's first prepare a set of folders on which we
will deploy our data so that it survives a reboot.

In Wiki VM's Terminal:

'''bash
cd /rw
sudo mkdir -p var/pgsql/data
sudo chown postgres:postgres var/pgsql/data
sudo chmod 700 var/pgsql/data
sudo mkdir -p var/pgsql/backups
sudo chown postgres:postgres var/pgsql/backups
sudo chmod 700 var/pgsql/backups
'''

### Initializing the database ###

Let's initialize the database cluster

'''bash
sudo su
rm -rf /var/lib/pgsql
ln -s /rw/var/pgsql /var/lib/pgsql
postgresql-setup initdb
exit
'''

### Configuring the database cluster ###

Let's make sure we can connect to the database cluster

Note: If you are new to linux a more natural text editor than [vi] can be used.
[nano] is a good one... replace vi with nano in all instructions. You just have
to know to use \[Ctrl\]+\[x\] to exit and save your file.

'''bash
sudo su - postgres
vi /var/lib/pgsql/data/pg_hba.conf
host confluence confluence 127.0.0.1/32 md5
host jira jira 127.0.0.1/32 md5
'''

Add the following lines toward the end of the file, below the IPv4 local
connections.

Note: If you wish to change the port, or make the database listen to another
address than the local loop 127.0.0.1, you will need to edit
pgsql/data/postgresql.conf and change the address above in
pgsql/data/pg_hba.conf.

Once done exit from the posgres sudo session

'''bash
exit
'''

### Configuring the jira database ###

Let's create the jira database user and the database

'''bash
sudo systemctl start postgresql
sudo -s -H -u postgres
/usr/bin/createuser -S -P -E jira
'''

Enter the password for the jira database user and take note of it. Once done
exit from the posgres sudo session

'''bash
/usr/bin/createdb --owner jira --encoding utf8 jira
'''

### Configuring the confluence database ###

Let's create the confluence database user and the database

'''bash
/usr/bin/createuser -S -P -E confluence
'''

Enter the password for the confluence database user and take note of it. Once
done exit from the posgres sudo session.

'''bash
/usr/bin/createdb --owner confluence --encoding utf8 confluence
exit
'''

### Preparing for a reboot ###

Let's now edit our rc.local script so that we set everything in place when the
VM reboots. In Wiki VM's Terminal:

'''bash
sudo vi config/rc.local
\#!/bin/bash
rm -rf /var/lib/pgsql
ln -s /rw/var/pgsql /var/lib/pgsql
/usr/bin/systemctl enable postgresql &
/usr/bin/systemctl start postgresql &
'''

Let's not forget to make this file executable

'''bash
sudo chmod u+x config/rc.local
'''

### Testing the databases ###

Stop the wiki VM

'''bash
sudo halt
'''

Start the VM by opening its Terminal.
Let's test our databases.
'''bash
sudo systemctl status postgresql
'''

This should give you a status of active (running).


Try to connect to the Jira database.

'''bash
psql -U jira -h 127.0.0.1 -p 5432 -d jira
'''

Note: Exit the database by typing "\q" and pressing \[Enter\]

Try to connect to the Confluence database.

'''bash
psql -U confluence -h 127.0.0.1 -p 5432 -d confluence
'''

Hopefully you are like me, all set for the next stage.

Preparing the Jira Tomcat instance
----------------------------------

### Preparing the read-write partition ###

In the read-write partition, let's first prepare a set of folders on which we
will deploy our application and data so that they survive a reboot. In Wiki VM's
Terminal:

'''bash
cd /rw
sudo mkdir -p opt/jira
sudo mkdir -p var/log/jira
sudo mkdir -p var/jira
'''

### Integrating with systemd ###

Looking into systemd tomcat's service config (/usr/lib/systemd/system/tomcat),
it states that to run a new instance of tomcat, we need to copy
/etc/sysconfig/tomcat to /etc/sysconfig/jira (jira being the name we want to
give to this tomcat instance) and set the variables we need. As the file will
not be persisted in our Qubes VM, let's prepare one that we will copy at boot
time.

'''bash
sudo mkdir config/sysconfig
sudo vi config/sysconfig/jira
CATALINA_BASE="/opt/jira"
CATALINA_TMPDIR="/opt/jira/temp"
TOMCAT_USER="jira"
CATALINA_PID="/var/run/jira.pid"
'''

We also have to provide a copy of /usr/lib/systemd/system/tomcat for ourservice.

Note: This copy needs to have SERVICE_NAME set prior to launching the tomcat
start-up script. As systemd mandate an absolute path for the command to be
executed, we are going to need to pass it via a shell command.

In Wiki VM's Terminal:

'''bash
sudo mkdir -p config/systemd/system
sudo cp /usr/lib/systemd/system/tomcat.service config/systemd/system/jira.service
'''

Let's modify the ExecStart, ExecStop, User and Group lines as follow

'''bash
sudo vi config/systemd/system/jira.service
ExecStart=/bin/bash -c 'export SERVICE_NAME="jira"; /usr/sbin/tomcat-sysd start'
ExecStop=/bin/bash -c 'export SERVICE_NAME="jira"; /usr/sbin/tomcat-sysd stop'
User=jira
Group=jira

### Preparing CATALINA_BASE ###

Let's prepare our CATALINA_BASE (/opt/jira), but we first need our jira account
set-up

'''bash
sudo chown jira:root var/log/jira
sudo chmod 770 var/log/jira
sudo chown jira:root var/jira
sudo chmod 770 var/jira
cd opt/jira
sudo mkdir -p conf/Catalina/localhost logs temp webapps work
sudo chown jira logs temp work
sudo chmod 770 logs temp work
'''

### Preparing for a reboot ###

Let's now edit our rc.local script so that we set everything in place when the
VM reboots

'''bash
cd /rw
sudo vi config/rc.local
'''

At the end of the file add the following

'''bash
rmdir /opt
ln -s /rw/opt /opt
ln -s /rw/var/jira /var/jira
ln -s /rw/var/log/jira /var/log/jira
cp /rw/config/sysconfig/jira /etc/sysconfig/
cp /rw/config/systemd/system/jira.service /usr/lib/systemd/system/
touch /var/run/jira.pid
chown jira:jira /var/run/jira.pid
/usr/bin/systemctl enable jira &
/usr/bin/systemctl start jira &
'''

Preparing the Confluence Tomcat instance
----------------------------------------

### Preparing the read-write partition ###

In the read-write partition, let's first prepare a set of folders on which we
will deploy our application and data so that they survive a reboot. In Wiki VM's
Terminal:

'''bash
sudo mkdir -p opt/confluence
sudo mkdir -p var/log/confluence
sudo mkdir -p var/confluence
'''

### Integrating with systemd ###

Looking into systemd tomcat's service config ('/usr/lib/systemd/system/tomcat'),
it states that to run a new instance of tomcat, we need to copy
'/etc/sysconfig/tomcat' to '/etc/sysconfig/confluence' (confluence being the
name we want to give to this tomcat instance) and set the variables we need.
As the file will not be persisted in our Qubes VM, let's prepare one that we
will copy at boot time.

'''bash
sudo vi config/sysconfig/confluence
CATALINA_BASE="/opt/confluence"
CATALINA_TMPDIR="/opt/confluence/temp"
TOMCAT_USER="confluence"
CATALINA_PID="/var/run/confluence.pid"
'''

We also have to provide a copy of /usr/lib/systemd/system/tomcat for our
service.

Note: This copy needs to have SERVICE_NAME set prior to launching the tomcat
start-up script. As systemd mandate an absolute path for the command to be
executed, we are going to need to pass it via a shell command.

In Wiki VM's Terminal:

'''bash
sudo cp config/systemd/system/jira.service config/systemd/system/confluence.service
'''

Let's modify the ExecStart, ExecStop, User and Group lines as follow:

'''bash
sudo vi config/systemd/system/confluence.service
ExecStart=/bin/bash -c 'export SERVICE_NAME="confluence"; /usr/sbin/tomcat-sysd start'
ExecStop=/bin/bash -c 'export SERVICE_NAME="confluence"; /usr/sbin/tomcat-sysd stop'
User=confluence
Group=confluence
'''

### Preparing CATALINA_BASE ###

Let's prepare our CATALINA_BASE (/opt/confluence), but we first need our
confluence account set-up.

'''bash
sudo chown confluence:root var/log/confluence
sudo chmod 770 var/log/confluence
sudo chown confluence:root var/confluence
sudo chmod 770 var/confluence
cd opt/confluence
sudo mkdir -p conf/Catalina/localhost logs temp webapps work
sudo chown confluence logs temp work
sudo chmod 770 logs temp work
'''

### Preparing for a reboot ###

Let's now edit our rc.local script so that we set everything in place when the
VM reboots.

'''bash
cd /rw
sudo vi config/rc.local
'''

At the end of the file add the following:

'''bash
ln -s /rw/var/confluence /var/confluence
ln -s /rw/var/log/confluence /var/log/confluence
cp /rw/config/sysconfig/confluence /etc/sysconfig/
cp /rw/config/systemd/system/confluence.service /usr/lib/systemd/system/
touch /var/run/confluence.pid
chown confluence:confluence /var/run/confluence.pid
/usr/bin/systemctl enable confluence &
/usr/bin/systemctl start confluence &
'''

Installing Jira
---------------

We have 2 potential way to proceed. Go with the WAR distribution and configure
Tomcat based on Fedora basic settings or go with the standalone distribution and
modify it to use our tomcat instance. I am going to fo with the second option as
the editor (Atlassian) has already done a good job at configuring it,
particularly in such a simple set up as the one I want to do.

Note: In the background I have looked at the differences between the two. I have
used a tool to remove html comments called [XMLStarlet] using the command
'xmlstarlet ed -d '//comment()' file.xml > file-clean.xml' and a very good
graphical diff program called [meld]. The Atlassian files are as expected better
one to use as a starting point with a few exceptions like conf/web.xml (which
contains more recent mime types).

### Downloading the standalone distribution ###

Go to [Atlassian Jira download page]

 * Toward the bottom of the page toggle the "All JIRA Download Options"
 * Download the Linux tar.gz file

### Copying the tar.gz file to Wiki VM ###

This  is documented in the [Qubes OS user documentation] page.

 * In Firefox menu select Tools/Downloads.
 * Right-Click on atlassian-jira-<version>.tar.gz and select Open Containing
   Folder.
 * Right-Click on the file and select Scripts / Copy to other AppVM.
 * input wiki as the destination domain name.
 * Allow the transfer to happen by clicking Yes

### Install Jira in the read-write partition ###

Let's expand our Jira distribution.

'''bash
cd
tar xzvf QubesIncoming/disp<X>/atlassian-jira-<version>.tar.gz
cd atlassian-jira-<version>-standalone
'''

Let's move into place the things we want.

'''bash
sudo mv conf/* /rw/opt/jira/conf
sudo chown -R root:root /rw/opt/jira/conf
sudo mv atlassian-jira /rw/opt/jira
sudo chown -R root:root /rw/opt/jira/atlassian-jira
sudo mv lib /rw/opt/jira
sudo chown -R root:root /rw/opt/jira/lib
sudo mv external-source /rw/opt/jira
sudo chown -R root:root /rw/opt/jira/external-source
sudo mv licenses /rw/opt/jira
sudo chown -R root:root /rw/opt/jira/licenses
sudo mv tomcat-docs /rw/opt/jira
sudo chown -R root:root /rw/opt/jira/tomcat-docs
'''

Let's make sure all parameters are past to the JVM. You can review this which is
what was in bin/setenv.sh which builds up JAVA_OPTS

'''bash
sudo vi /rw/config/sysconfig/jira
'''

Add the following at the begining of the file:

'''
#
#  Occasionally Atlassian Support may recommend that you set some specific JVM a rguments.  You can use this variable below to do that.
#
JVM_SUPPORT_RECOMMENDED_ARGS=""

#
# The following 2 settings control the minimum and maximum given to the JIRA Jav a virtual machine.  In larger JIRA instances, the maximum amount will need to be increased.
#
JVM_MINIMUM_MEMORY="384m"
JVM_MAXIMUM_MEMORY="768m"
 
#
# The following are the required arguments for JIRA.
#
JVM_REQUIRED_ARGS="-Djava.awt.headless=true -Datlassian.standalone=JIRA -Dorg.apache.jasper.runtime.BodyContentImpl.LIMIT_BUFFER=true -Dmail.mime.decodeparameters=true -Dorg.dom4j.factory=com.atlassian.core.xml.InterningDocumentFactory"
 
# Perm Gen size needs to be increased if encountering OutOfMemoryError: PermGen problems. Specifying PermGen size is not valid on IBM JDKs
JIRA_MAX_PERM_SIZE=384m
'''

You should have then this...

'''
CATALINA_BASE="/opt/jira"
CATALINA_TMPDIR="/opt/jira/temp"
TOMCAT_USER="jira"
CATALINA_PID="/var/run/jira.pid"
'''

Add the following at the end:

'''
#-----------------------------------------------------------------------------------
#
# In general don't make changes below here
#
#-----------------------------------------------------------------------------------
JVM_EXTRA_ARGS="-XX:+PrintGCDateStamps -XX:-OmitStackTraceInFastThrow"
 
JAVA_OPTS="-XX:MaxPermSize=${JIRA_MAX_PERM_SIZE} -Xms${JVM_MINIMUM_MEMORY} -Xmx${JVM_MAXIMUM_MEMORY} ${JAVA_OPTS} ${JVM_REQUIRED_ARGS} ${DISABLE_NOTIFICATIONS} ${JVM_SUPPORT_RECOMMENDED_ARGS} ${JVM_EXTRA_ARGS}
'''

OK, let's clean the dust up.

'''bash
cd
rm -rf atlassian-jira-<version>-standalone
'''

### Set the Jira Home directory ###

Let's set our JIRA Home Directory.

'''bash
sudo vi /rw/opt/jira/atlassian-jira/WEB-INF/classes/jira-application.properties
jira.home = /var/jira
'''

### Fix the context ###

There is a small bug (or relaxed configuration as it is a standalone distrib)
in the Atlassian package as the server.xml is referring to catalina.home instead
of catalina.base.

'''bash
sudo vi /rw/opt/jira/conf/server.xml
replace the docBase value from ${catalina.home} to ${catalina.base}
'''

Note: While you are at it, you may want to also change the ports the server is
listening on.

### Prepare Jira for reverse proxy aware responses ###

#### Set the context path ####

'''bash
sudo vi /rw/opt/jira/conf/server.xml
'''

Replace the following line:

'''xml
<Context path="" docBase="${catalina.base}/atlassian-jira" reloadable="false" useHttpOnly="true">
'''

With:

'''xml
<Context path="/jira" docBase="${catalina.base}/atlassian-jira" reloadable="false" useHttpOnly="true">
'''

#### Set the URL for redirection ####

'''bash
sudo vi /rw/opt/jira/conf/server.xml
'''

Replace the following line:

'''xml
<Connector port="8080" maxThreads="150" minSpareThreads="25" connectionTimeout="20000" enableLookups="false" maxHttpHeaderSize="8192" protocol="HTTP/1.1" useBodyEncodingForURI="true" redirectPort="8443" acceptCount="100" disableUploadTimeout="true"/>
'''

With:

'''xml
<Connector port="8080" maxThreads="150" minSpareThreads="25" connectionTimeout="20000" enableLookups="false" maxHttpHeaderSize="8192" protocol="HTTP/1.1" useBodyEncodingForURI="true" redirectPort="8443" acceptCount="100" disableUploadTimeout="true" proxyName="www.example.com" proxyPort="443" scheme="https"/>
'''

Which set the name of the external URL, its port and protocol so that content
generated by Tomcat contains links in the form 'https://www.example.com/<blah>'.

Installing Confluence
---------------------

### Downloading the standalone distribution ####

Go to [Atlassian Confluence download page]

 * Download the Standalone Linux tar.gz file

### Copying the tar.gz file to Wiki VM ###

This  is documented in the [Qubes OS user documentation] page.

 * In Firefox menu select Tools/Downloads.
 * Right-Click on atlassian-confluence-<version>.tar.gz and select Open Containing Folder.
 * Right-Click on the file and select Scripts / Copy to other AppVM.
 * input wiki as the destination domain name.
 * Allow the transfer to happen by clicking Yes

### Install Confluence in the read-write partition ###

Let's expand our Confluence distribution.

'''bash
cd
tar xzvf QubesIncoming/disp<X>/atlassian-confluence-<version>.tar.gz
cd atlassian-confluence-<version>
'''

Let's move into place the things we want.

'''bash
sudo mv conf/* /rw/opt/confluence/conf
sudo chown -R root:root /rw/opt/confluence/conf
sudo mv confluence /rw/opt/confluence
sudo chown -R root:root /rw/opt/confluence/confluence
sudo mv lib /rw/opt/confluence
sudo chown -R root:root /rw/opt/confluence/lib
sudo mv licenses /rw/opt/confluence
sudo chown -R root:root /rw/opt/confluence/licenses

Let's make sure all parameters are past to the JVM. You can review this which is
what was in 'bin/setenv.sh' which builds up 'JAVA_OPTS'

'''bash
sudo vi /rw/config/sysconfig/confluence
'''

Add the following at the end of the file:

'''
JAVA_OPTS="-Xms256m -Xmx512m -XX:MaxPermSize=256m $JAVA_OPTS -Djava.awt.headless=true "
'''

And clean the dust up.

'''bash
cd
rm -rf atlassian-confluence-<version>

### Set the Confluence Home directory ###

Let's set our Confluence Home Directory.

'''bash
sudo vi /rw/opt/confluence/confluence/WEB-INF/classes/confluence-init.properties
confluence.home = /var/confluence
'''

### Prepare Confluence for reverse proxy aware responses ###

#### Set the context path ####

'''bash
sudo vi /rw/opt/confluence/conf/server.xml
'''

Replace the following line:

'''xml
<Context path="" docBase="../confluence" debug="0" reloadable="false" useHttpOnly="true">
'''

With:

'''xml
<Context path="/confluence" docBase="../confluence" debug="0" reloadable="false" useHttpOnly="true">
'''

#### Set the URL for redirection ####

'''bash
sudo vi /rw/opt/jira/conf/server.xml
'''

Replace the following line:

'''xml
<Connector className="org.apache.coyote.tomcat4.CoyoteConnector" port="8090" minProcessors="5" maxProcessors="75" enableLookups="false" redirectPort="8443" acceptCount="10" debug="0" connectionTimeout="20000" useURIValidationHack="false" URIEncoding="UTF-8"/>
'''

With:

'''xml
<Connector className="org.apache.coyote.tomcat4.CoyoteConnector" port="8090" minProcessors="5" maxProcessors="75" enableLookups="false" redirectPort="8443" acceptCount="10" debug="0" connectionTimeout="20000" useURIValidationHack="false" URIEncoding="UTF-8" proxyName="www.example.com" proxyPort="443" scheme="https"/>
'''

Which set the name of the external URL, its port and protocol so that content
generated by Tomcat contains links in the form 'https://www.example.com/<blah>'.

Preparing the reverse proxy
---------------------------

In order to present both applications under the same web server and to map the
ports to a default port, we are going to use [mod_proxy] and [mod_proxy_http].
For more information on how to set-up a more complex set-up you may want to look
at the [Atlassian page]. We will not use [mod_proxy_connect] as we will decrypt
the connection on our reverse proxy and forward the traffic unencrypted to
tomcat.

Note: If you are planning on hosting other web applications than Jira and
Confluence, you may prefer to host the reverse proxy in a dedicated VM which
will forward traffic to different web application back end VMs.

Note2: This set-up does not provide caching due to the small expected number of
users and the increased attack surface.

### Preparing the read-write partition ###

In the read-write partition, let's first prepare a set of folders on which we
will deploy our application and data so that they survive a reboot.In Wiki VM's
Terminal:

'''bash
cd /rw
sudo mkdir -p config/httpd/conf
sudo mkdir -p config/httpd/conf.d
sudo mkdir -p config/httpd/conf.modules.d
'''

### Configuring httpd ###

Let's prepare our httpd.conf file.

'''bash
sudo cp /etc/httpd/conf/httpd.conf config/httpd/conf
sudo vi config/httpd/conf/httpd.conf
'''

Comment out the following line:

'''
Listen 80
'''

Change the following line:

'''
ServerAdmin root@localhost
'''

Add the following lines:

'''
ServerName www.example.com
'''

The proxy load module file /etc/httpd/conf.modules.d/00-proxy.conf is already
loading everything.Let's prepare the associated conf file:

'''bash
sudo vi config/httpd/conf.d/00-proxy.conf
ProxyRequests Off
ProxyPreserveHost On
<Proxy *>
    Order deny,allow
    Allow from all
</Proxy>
 
<Location /confluence>
    Order allow,deny
    Allow from all
    ProxyPass http://127.0.0.1:8090/confluence
    ProxyPassReverse http://127.0.0.1:8090/confluence
    SetEnv proxy-sendchunks 1
    SetEnv proxy-interim-response RFC
    SetEnv proxy-initial-not-pooled 1
</Location>

<Location /jira>
    Order allow,deny
    Allow from all
    ProxyPass http://127.0.0.1:8080/jira
    ProxyPassReverse http://127.0.0.1:8080/jira
    SetEnv proxy-sendchunks 1
    SetEnv proxy-interim-response RFC
    SetEnv proxy-initial-not-pooled 1
</Location>
'''

### Configuring openssl ###

Let's start by adjusting the SSL configuration file.

'''bash
sudo cp /etc/httpd/conf.d/ssl.conf config/httpd/conf.d
sudo vi config/httpd/conf.d/ssl.conf
SSLProtocol TLSv1.2
SSLCipherSuite ECDHE-ECDSA-AES128-SHA256:DHE-RSA-AES128-SHA
SSLHonorCipherOrder on
'''

To enable TLSv1.2 in Firefox:

 * browse to "about:config"
 * accept the risk
 * search for tls and set security.tls.version.max=3

/!\ Under construction /!\|!| Note: Road blocked.|!|

\!/      After this sign     \!/


Let's generate our private key. I will not get it signed as I do not have a
fixed IP address and do not want to trust Certificates Authorities.

'''bash
sudo yum install crypto-utils
genkey www.example.com
'''

Note: I have used a 4096bits key without certificate request or passphrase.

Note2: There is a bug in fedora broken by upstream package for self signed
certificate (as reliant at the moment on md5).


/!\ Under construction /!\|!| Note: Road blocked.|!|

\!/      Before this sign     \!/

### Preparing for a reboot ###

Let's now edit our rc.local script so that we set everything in place when the
VM reboots.

'''bash
sudo vi config/rc.local
'''

At the end of the file add the following:

'''bash
mv /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.origin
cp /rw/config/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf
cp /rw/config/httpd/conf.d/00-proxy.conf /etc/httpd/conf.d
cp /rw/config/httpd/conf.d/ssl.conf /etc/httpd/conf.d
/usr/bin/systemctl enable httpd &
/usr/bin/systemctl start httpd &
'''

If you want to be able to access Jira and Confluence from another browser than
the wiki VM one, add the following line where 10.137.2.x is the wiki VM's
IPAddress of eth0 (ifconfig | grep -i ast) and y.y.y.y/z is the subnet you want
to allow in.

'''bash
/usr/sbin/iptables -I INPUT 5 -j ACCEPT -d 10.137.2.x -s y.y.y.y/z -p tcp --dport 443 -m state --state NEW
'''

You need to have www.example.com configured in your DNS. If you prefer to do
that later you can add the following line to allow you to continue the
configuration via the wiki VM firewall without the DNS entry set.

'''bash
echo 10.137.2.x www.example.com >> /etc/hosts
'''

Note: becareful to have two superior sign so that the line is appended to the
file.

Testing
-------

### Testing the httpd service ###

Stop the wiki VM.

'''bash
sudo halt
'''

Start the VM by opening its Firefox browser and browse to
'https://www.example.com'

You should get a warning that "This Connection is Untrusted". This is because we
have a self signed certificate.

 * Click on I Understand the Risks
 * Add Exception...
 * Confirm Security Exception...

You should see the Fedora Apache test page.

Note: If you get a "Secure Connection Failed", remember that you have to enable
TLSv1.2 in Firefox as mentioned above.

### Troubleshooting httpd ###

If you have issues. Troubleshoot meticulously. Let's first do very basic check
on the server side.

#### Apache httpd service is up ####

Check the service is started:

'''bash
sudo systemctl status httpd
'''

#### Apache httpd service is listening on the right port ####

Check the service is listening on the right port (443)

'''bash
netstat -an | grep LIST | grep -v STREAM
'''

#### Browser proxy settings ####

Your browser will use your browser proxy settings and either go direct or via
proxy.

#### DNS resolution ####

If the browser is going via a proxy, the proxy will do the dns resolution,
otherwise, the first thing your browser does is a dns query to the name you
entered in the browser destination bar.

Make sure you can resolve this name using a command line tool such as nslookup
or dig.

If it can't check you /etc/hosts file then your /etc/resolv.conf file where your
dns server is defined.

#### TCP Connection ####

The next thing your browser (or proxy) will do is establish a TCP connection on
port 443.This should give you a status of active (running).


Make sure you can connect over TCP to the web server by issuing telnet
'10.137.2.<x> 443'. you should get a prompt. If you don't, you can install a
tool such as tcptraceroute to help identify where routing is broken or where a
firewall is in the way.

#### SSL Handshake ####

If you have gone this far you should be OK unless there is a mismatch between
the Protocol and Cipher supported by the web server and your browser. You can
check the protocols you browser support by going to this
[Hannover's university web site].

If you manage to connect you can view which cipher suite you used by looking in
the httpd logs:

'''bash
sudo su
tail /var/log/httpd/ssl_request_log
exit
'''

### Testing the Jira and Confluence tomcat instances ###

Browse to Jira and Confluence

'''bash
https://www.example.com/jira
https://www.example.com/confluence
'''

After a little time you should see the Jira Welcome page and the Confluence
Setup Wizard page.

### Troubleshooting Jira and Confluence tomcat instances ###

#### Jira tomcat instance is up ####

Check the service is started

'''bash
sudo systemctl status jira 
'''

In case of issues check the logs:

'''bash
sudo cat /opt/jira/logs/catalina.out
'''

#### Jira tomcat instance is visible through the reverse proxy ####

Check that your request is hitting httpd:

'''bash
sudo su
tail /var/log/httpd/ssl_request_log
tail /var/log/httpd/ssl_access_log
tail /var/log/httpd/ssh_error_log
exit 
'''

Check that your request is hitting Jira tomcat instance:

'''bash
sudo su
tail /opt/jira/logs/access_log.<date>
exit
'''

The first you should look for is a 302 response for /jira redirecting you.

Which should therefore be followed by another 302 response for /jira/
redirecting you again.

Which should then be followed by a 200 response for
/jira/secure/SetupDatabase!default.jspa.

If you are missing the 302 to /jira/ you may have an incorrect Connector set-up
in /opt/jira/conf/server.xml. Verify your proxyName, proxyPort and scheme
values. They should be so that whatever the server is sending to the client
allows the client to connect back.

Configuring Jira and Confluence
-------------------------------

### Configuring Jira ###

Open the wiki VM's Firefox browser and browse to:
'https://www.example.com/jira'

 * Select the server Language.
 * Select My Own Database as the Database Connection
 * Select PostgreSQL as the Database Type
 * Input 127.0.0.1 as the Hostname
 * Input 5432 as the Port
 * Input jira as the Database
 * Input jira as the Username
 * Input the jira's database password as Password
 * Input public as the Schema
 * Click on Test Connection
 * Click on Next

It will take some time (a couple of minutes for me) for the database and Jira
to be set-up, be patient.

 * Input an Application Title
 * Select the Mode
 * Verify the Base URL (https://www.example.com/jira)

Note: I recommend you get the 10 users licences. The trial licence is for
several hundreds of users, I suspect there is no issue in going for the 10 users
licences later, but I haven't tested this.

 * Select the I have a licence option

Open a Disposable VM Firefox Browser and go to <https://my.atlassian.com/>
to retrive it.

 * Copy the licence from the disposable VM to the wiki VM browser.

After some time you will be prompted to enter the details of the administrator
account.

### Configuring Confluence ###

Open the wiki VM's Firefox browser and browse to:

 * https://www.example.com/confluence
 * Copy the licence from the disposable VM to the wiki VM browser.
 * Select the external database option
 * select the JDBC connector option

Setup the Database

 * Verify org.postsgresql.Driver is the Driver Class Name
 * Verify jdbc:postgresql://localhost:5432/confluence is the Database URL
 * Input confluence as the User Name
 * Input the confluence's database password as the Password
 * Click Next

[Jira]: https://www.atlassian.com/software/jira
[Confluence]: https://www.atlassian.com/software/confluence
[Tomcat]: http://tomcat.apache.org/
[PostgreSQL]: http://www.postgresql.org/
[apache httpd]: http://httpd.apache.org/
[QubesOS]: http://qubes-os.org/
[trac]: http://trac.edgewall.org/wiki/TracInstall
[Qubes user group]: https://groups.google.com/forum/#!topic/qubes-users/2gHjwj3YRrI
[vi]: http://www.vim.org/
[nano]: http://www.nano-editor.org/
[XMLStarlet]: http://xmlstar.sourceforge.net/
[meld]: http://meldmerge.org/
[Atlassian Jira download page]: https://www.atlassian.com/software/jira/download
[Qubes OS user documentation]: http://qubes-os.org/trac/wiki/UserDoc
[Atlassian Confluence download page]: https://www.atlassian.com/software/confluence/download
[mod_proxy]: http://httpd.apache.org/docs/2.4/mod/mod_proxy.html
[mod_proxy_http]: http://httpd.apache.org/docs/2.4/mod/mod_proxy_http.html
[Atlassian page]: https://confluence.atlassian.com/display/DOC/Using+Apache+with+mod_proxy
[mod_proxy_connect]: http://httpd.apache.org/docs/2.4/mod/mod_proxy_connect.html
[Hannover's university web site]: https://cc.dcsec.uni-hannover.de/
