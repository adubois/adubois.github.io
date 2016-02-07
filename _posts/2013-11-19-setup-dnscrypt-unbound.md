---
layout: post
title:  "How-to set up dnscrypt-proxy and unbound on Qubes OS"
date:   2013-11-19 21:03:44 +0000
categories: howto
---

This article describes how to set up dnscrypt-proxy v1.3.3 and unbound 1.4.32
on Qubes OS v2 Beta 2.

[dnscrypt-proxy] is a tool which will encrypt DNS queries between the client and
the resolver. It relies on libsodium which is an encryption library trying to
make use of only well respected algorithms by the cryptographic community.

[unbound] is a is a validating, recursive, and caching DNS resolver.

[QubesOS] is at the time of writing, to my view, one of the most secure Linux
based desktop operating system. It can run Linux and Windows guest virtual
machines and isolate VMs, networking, usb and graphical stacks levering on Xen
hypervisor. It also leverage on a templating system to minimize the disk
footprint of VMs based on the same template. It does not support 3D graphical
operations out of the box but advanced users can set-up a second Video card and
dedicate it to one of their VM.

The aim of this post is to allow new QubesOS users to set-up a DNS service VM
which will serve DNS queries for their VMs while ensuring encryption of the
communication between their desktop and the DNS provider of their choice which
support dnscrypt. unbound will be use to cache DNS queries and serve them to
your VMs.
Discussion about this post in the QubesOS user group can be found
[here][QubesOS user group post]

Preparing the Template
----------------------

### Installing Unbound ###

Launch the terminal in the Template VM.

 * Click Application Menu (top left)
 * Select Template fedora-18-x64 / fedora-18-x64: Terminal

Install the Unbound Server by typing in the template's Terminal:

```bash
sudo yum install unbound
```

For these changes to be visible to the DNS VM, shutdown the Template VM.

```bash
sudo halt
```

Setting up dnscrypt-proxy on a DNS VM
-------------------------------------

### Creating a DNS VM ###

In QubesOS VM Manager:

 * Select in the menu VM / Create AppVM
 * Give it the name dns
 * Select the color yellow
 * Launch the creation by pressing OK

Note: The attack surface of this VM will be fairly small. We have therefore used
the yellow color.

Attacks could be launched by malformed DNS queries (from an untrusted VM aiming
at compromising the DNS service data, i.e. Cache poisoning, or service) or
responses (from a DNS provider). To mitigate the first risk, you may want to
implement a trusted dnsVM and an untrusted dnsVM aimed at serving VM of
different level of trust in you QubesOS VMs setup.

Attacks could also be launched by a man in the middle (external network, or
possibly from the netVM) by attacking the decryption stack that we will use,
libsodium, to either gain visibility on the data transmitted or possibly
compromising the dns service.

Overall you may want to use different client stack for the dns resolution by the
dnsVM and your other AppVMs so that a vulnerability in one cannot be used to
swim back to initial requester.

### Starting the DNS VM ###

Launch the terminal in the dnsVM. In the Application Menu:

 * Select the Domain: dns / dns: Terminal

### Installating the development tools ###

In QubesOS, it is usually preferable to install software packages in the
template VM from which the VM you will use the tool from is based on. In this
case, these tools will only be required to compile 2 packages. It is therefore
preferable to install them in the dnsVM. They will not be available any more
after a dnsVM reboot, but we will have make sure everything is OK before
proceeding.

In dns: Terminal

```bash
sudo yum groupinstall "Development Tools"
```

### Downloading the source code ###

Note: The most detailed information on how to install dnscrypt-proxy was
available on github.

Let's launch a web browser in a disposable VM and download our source...

In Application Menu:

 * Select DisposableVM / DispVM: Firefox web browser.
 * Browse to <https://dnscrypt.org>
 * Click on "Download packages" or go to <https://dnscrypt.org/dnscrypt-proxy/downloads>
 * Download the lastest version and save the file.
 * Browse to [Libsodium]
 * Click on "Download the tarball" or go to <https://download.libsodium.org/libsodium/releases/>
 * Download the lastest version and save the file.

### Copying the source code to the dns VM ###

This  is documented in the [QubesOS user documentation page].

 * In Firefox menu select Tools/Downloads.
 * Right-Click on dnscrypt-proxy-<version>.tar.bz2 and select Open Containing Folder.
 * Right-Click on the file and select Scripts / Copy to other AppVM.
 * input dns as the destination domain name.
 * Allow the transfer to happen by clicking Yes
 * Repeat the operation for the file libsodium-<version>.tar.gz.

### Verifying the source ###

In the dns VM Terminal:

Calculate the hash of dnscrypt-proxy:

```bash
openssl dgst -sha256 ~/QubesIncoming/dispXX/dnscrypt-proxy-<version>.tar.bz2
```

In a new Disposable VM:

Launch terminal

 * Select your new disposable VM in QubesOS VM Manager
 * Right-click / Run command in VM
 * 'terminal# as the command to execute

Retrieve the original hash using DnsSec:

```bash
dig +short +dnssec TXT dnscrypt-proxy-<version>.tar.bz2.download.dnscrypt.org
```
 
Compare the 2 hashes, they must be identical.

Repeat the process for the second file but this time check against
'dig +dnssec +short TXT <file>.download.libsodium.org'

### Building the source ###

Let's start with Libsodium. In dns VM Terminal:

```bash
cd Documents
mkdir libsodium
cd libsodium
tar xzvf ~/QubesIncoming/disp<XX>/libsodium-<version>.tar.gz
cd libsodium-<version>
./configure
make -j<numberOfCoresOnYourSystem>
sudo make install
```

As we use a RedHat based system, we need to make sure our libraries are linked.
In the dns VM Terminal:

```bash
sudo su -
echo /usr/local/lib > /etc/ld.so.conf.d/usr_local_lib.conf
ldconfig
exit
```

Let's build dnscrypt-proxy now. In dns VM Terminal:

```bash
cd Documents
mkdir dnscrypt-proxy
cd dnscrypt-proxy
tar xjvf ~/QubesIncoming/disp<XX>/dnscrypt-proxy-<version>.tar.bz2
```

Note: We used the j option as the compression library used by this source is
different...

```bash
cd dnscrypt-proxy-<version>
./configure
make -j<numberOfCoresOnYourSystem>
sudo make install
```

### Configuring the start up script ###

In dns VM Terminal:

```bash
cd /rw/config
sudo chmod u+x rc.local
sudo vi rc.local
#!/bin/bash
/bin/echo /usr/local/lib > /etc/ld.so.conf.d/usr_local_lib.conf
/sbin/ldconfig
/usr/local/sbin/dnscrypt-proxy --provider-key=B735:1140:206F:225D:3E2B:D822:D7FD:691E:A1C3:3CC8:D666:8D0C:BE04:BFAB:CA43:FB79 --provider-name=2.dnscrypt-cert.opendns.com. --resolver-address=208.67.220.220:443 --daemonize
```

Note: I have not yet been able to start the daemon as a low privileged user
(tried to reuse tinyproxy).

### Testing the daemon ###

Before rebooting the VM and loosing our development tools, let's check
everything went according to plan... In dns VM Terminal:

```bash
cd /rw/config
sudo ./rc.local
nslookup
server 127.0.0.1
www.opendns.com
```

You should get the IPAddress of opendns.com web site...

Note: If after a reboot you try again the nslookup and is does not work, you
probably forgot to update the libraries using ldconfig as described above.

Installing Unbound
------------------

### Preparing the read-write partition ###

In the read-write partition, let's first prepare a set of folders on which we
will deploy our configuration files so that they survive a reboot. In DNS VM's
Terminal:

```bash
cd /rw
sudo mkdir -p config/unbound/conf.d
sudo mkdir -p config/unbound/local.d
sudo cp /etc/unbound/unbound.conf config/unbound
```

### Configuring Unbound ###

Let's configure Unbound so that:

 * It listen to our eth0 interface IP Address ('ifconfig | grep -i ast')
 * It control who can do DNS queries
 * It host a local zone
 * And forward to dns-crypt the rest...

```bash
sudo vi config/unbound/unbound.conf
```

add the lines:

```bash
interface: 10.137.2.x
access-control: 10.137.2.0/24 allow
access-control: 10.138.2.0/24 allow
access-control: x.x.x.x/y allow
harden-large-queries: yes
private-address: 10.0.0.0/8
private-address: 192.168.0.0/16
val-permissive-mode: yes
do-not-query-localhost: no
```

Let's now create our local zone:

```bash
sudo vi config/unbound/local.d/local.conf
local-zone: "my-intranet.com" transparent
local-data: "www.my-intranet.com A x.x.x.x"
local-data: "my-share.my-intranet.com A x.x.x.x"
```

Finally let's make sure we forward all non served locally requests to dns-crypt

```bash
forward-zone:
    name: "."
    forward-addr: 127.0.0.1@53
```

Your rc.local script should look like this:

```bash
#!/bin/bash

/bin/echo /usr/local/lib > /etc/ld.so.conf.d/usr_local_lib.conf
/sbin/ldconfig
/usr/local/sbin/dnscrypt-proxy --provider-key=B735:1140:206F:225D:3E2B:D822:D7FD:691E:A1C3:3CC8:D666:8D0C:BE04:BFAB:CA43:FB79 --provider-name=2.dnscrypt-cert.opendns.com --resolver-address=208.67.220.220:443 --daemonize
/usr/bin/sleep 2

# Setting-up and starting unbound
/usr/bin/cp /rw/config/unbound/unbound.conf /etc/unbound/
/usr/bin/cp /rw/config/unbound/local.d/local.conf /etc/unbound/local.d/
/usr/bin/systemctl start unbound &
/usr/bin/sleep 2
/usr/sbin/iptables -I INPUT 3 -j ACCEPT -d 10.137.2.x -p udp --sport 1024:65535 --dport 53 -m conntrack --ctstate NEW
/usr/sbin/iptables -I INPUT 3 -j ACCEPT -d 10.137.2.x -p tcp --sport 1024:65535 --dport 53 -m conntrack --ctstate NEW
```

[dnscrypt-proxy]: http://dnscrypt.org/
[unbound]: http://unbound.net/
[QubesOS]: http://qubes-os.org/
[QubesOS user group post]: https://groups.google.com/forum/#!topic/qubes-users/-WvPZwE4CRc
[Libsodium]: https://github.com/jedisct1/libsodium
[QubesOS user documentation page]: http://qubes-os.org/trac/wiki/UserDoc
