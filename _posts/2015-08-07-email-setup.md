---
layout: post
title: How I set up my email
date: 2015-08-07 22:50
categories: blog
tags: [email, linux, howto, offlineimap, mutt, msmtp]
---
Since we released the 
[OpenPower Firmware](http://github.com/open-power/op-build) a year ago, I've
been spending a lot more time on github and mailing lists.  I was finding that
my combination of gmail and Lotus Notes was getting cumbersome.  I've had a
few cases where a mailing list I was responding to stripped out my messages
because Notes created my email as an HTML attachement.  So, I've decided to
setup an entirely command-line email environment for any open source work I
do.  I arrived at using:

* [mutt](www.mutt.org)
* [msmtp](msmtp.sourceforge.net)
* [offlineimap](offlineimap.org)
<!--more-->

### Domain Config ###
I wanted something more unique than just another gmail address, so I
registered a domain at [nic.xyz](nic.xyz) and routed all the email to
[zoho.com](zoho.com). Per this instructions on zoho, this requires setting up 
two MX records as follows:

| Hostname | Address      | Priority |
|:--------:|--------------|:--------:|
| @        | mx.zoho.com  | 10 |
| @        | mx2.zoho.com | 20 |

### IMAP ###
`offlineimap` provides a way to synchronize a remote IMAP server into a local
maildir directory for your email client to interface with.  The main advantage
of this is the "offline" part of it; you can view all of your email when disconnected.  The following is my `~/config/offlineimap/config` file:

~~~
[general]
accounts = username@domain

[Account username@domain]
localrepository = local-username@domain
remoterepository = remote-username@domain

[Repository local-username@domain]
type = Maildir
localfolders = ~/Mail/username@domain

[Repository remote-username@domain]
type = IMAP
remotehost = imap.zoho.com
remoteuser = username@domain.xyz
remotepass = xxxxxxxx
ssl = yes
sslcacertfile = /etc/ssl/certs/ca-certificates.crt
~~~

I also asked `offlineimap` to synchronize every 5 minutes by adding it to my
crontab.

~~~
*/5 * * * * offlineimap -u quiet
~~~

### SMTP ###
In order to send email, you need an MTA (Mail Transfer Agent) installed.  This
is where I had the most difficulty.  Initially my system had `postfix`
installed, of why I have no idea, but the configuration of it seemed
overwhelming.  I briefly tried `ssmtp` but quickly discovered it had trouble
with TLS and didn't support multiple accounts.  Then I tried `msmtp`, which
is light-weight, supports multiple accounts and TLS, and has consumable
documentation.

I have three email addresses I use regularly on my system.  My personal gmail
account, my work account, and this new one.  Since I use git for most of my
development and I infrequently send patches via email, I wanted something that
I can get `git-send-email` to use.  Specifically, I wanted to automatically
use the correct SMTP server based on the `user.email` setting of the git
repository I was mailing from.  `msmtp` helps this to happen.

The following is my `~/.msmtprc` file that configures three different email
accounts.

~~~
defaults

auto_from on
tls_trust_file /etc/ssl/certs/ca-certificates.crt

account domain
host smtp.zoho.com
port 587
tls on
maildomain domain.xyz
from username@domain.xyz
auth on
user username@domain.xyz
password password

account gmail
host smtp.gmail.com
port 587
tls on
maildomain gmail.com
from username@gmail.com
auth on
user username@gmail.com
password password

account work
host smtprelay.work.com
tls off
port 25
maildomain work.com
from username@work.com
auth off

account default : domain
~~~

Getting git to utilize this three account setup only requires adding the
following to your `~/.gitconfig`:

~~~
[sendemail]
    smtpServer = /usr/bin/msmtp
    envelopeSender = auto
~~~

### Mutt ###
Getting `mutt` to send and receive email was pretty straight-forward.  You
need to point `mutt` at the `offlineimap` mail directory set up earlier and
tell it to use `msmtp` instead of `sendmail`.  I also added key bindings to
switch the 'From' address between my three accounts and to force 
`offlineimap` to perform a synchronization.

~~~
set mbox_type=Maildir
set folder = ~/Mail/username@domain/

set sendmail="/usr/bin/msmtp"
set envelope_from=yes

macro generic "<esc>1" ":set from=username@domain.xyz"
macro generic "<esc>2" ":set from=username@gmail.com"
macro generic "<esc>3" ":set from=username@work.com"
macro generic S "<shell-escape>offlineimap\r"
~~~

### Summary ###
And that's it.  Every 5 minutes `offlineimap` fetches any new mail into a
new directory.  `git-send-email` and `mutt` can both send mail from any of
my accounts and `msmtp` routes it to the correct SMTP relay server.
