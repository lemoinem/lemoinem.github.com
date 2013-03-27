---
layout: post
title: "\"Migrating\" (Rescuing) a Mac OS X server to a Linux-based server"
---

\[Disclaimer: Here is another version of a blog-post I wrote for my company
blog. You can find the corporate version at
<http://www.mentel.com/migrating-a-mac-os-x-server-to-a-linux-based-server/>. They
kindly allow me to publish a copy on here!\]

[As said
previously](http://blog.mlemoine.name/2013/03/20/deploying-ruby-app-in-an-hostile-environment.html),
Murphy's Law is never too far so, "Sh\*t happens!”. And here is the best example
I can think of up to now: Place yourself any other Monday morning, it's about
7:30am (no, not [4am](http://www.ted.com/talks/rives_on_4_a_m.html), although
that's probably the moment it all started...). You are barely emerging from deep
sleep, when your cellphone rings:

* \[Great, that's gonna be a good week, and it's the job! Even better, what
  could've possibly happened! No email from the monitoring system though, so it
  couldn't be anything too bad...\]
* *Taking the call* "Yeah, Hello?"
* "Mathieu? Good morning! Hey, we have a little situation here \[Uh oh,
  "little"?...\]: The Mac OS X Server was crashed when I came in \[Cr\*p!\] and I
  rather like to keep it Off \[???\] because it's so noisy I think it's going to
  take off \[WTHeck!!\] and I'm pretty sure I'll see some smoke \[That's it I'm
  dreaming and will wake up for real soon\]..."
* "OK, on my way" \[What else would you have replied?\]...

Okay, to really get you a good picture how deep in it we are, a few things to
know about this server:

1. It is the only Mac OS X server in a Linux based network architecture. So it's
  already a little piece of Hell of compatibility just by itself.
2. It is (supposed to be) backed up via `Time Machine` on an external drive
  (Good News!)
3. It is our DHCP/DNS/OpenLDAP (read OpenDirectory) server.


Meaning: It is one of these temporarily definitive servers put in place when the
company was using less than 10 Mac OS X workstation and a NAS and that
generations of sysadmin meant to replace "Real Soon Now"™ but never had on top
of their priority list (If it ain't broken...). There is no graceful fallback
and it is impossible to have a drop-in replacement although we're supposed to
have back ups. Finally, while PROD is not impacted (Yeah!), nobody in the office
currently have an Internet access or even an IP and some are even locked out of
their workstations or vital internal services.

We were already in the middle of a slow-motion no-downtime migration of our
internal servers from a joyful mix of Centos 5.5+ / Debian Lenny/Squeeze /
Windows / virsh-kvm-qemu/ESXi to something a little more homogeneous (mostly
based on Ubuntu LTS). This process was going rather smoothly so we only have to
worry about Linux servers (Windows and ESXi were, fortunately, already out of
the picture), we already have a preliminary migration plan for the LDAP
(<http://blog.mlemoine.name/2012/09/07/migrate-mac-os-x-10.6-open-directory-to-unix-open-ldap-including-passwords.html>),
and the mandatory spare hardware to replace or late dinosaur is available thanks
to previously decommissioned servers. That's the bright side of life right here!

A taxi ride later, I realize the last actual `Time Machine` backup is from about
a month ago which is actually pretty good! Our DHCP has been stable since the
Genesis and our internal DNS don't change that often either. Our renewed server
farmed being based mostly on Ubuntu LTS (12.04.1 as of today), a couple of
software migration has to be done. We now use the standard ISC's DCHPd and bind9
as DNS. Since the former's configuration is straightforward, we didn't even need
to look at the TimeMachine to restore it \[1/3!\]. We, however, have a surprise
when trying to restore the DNS configuration, even putting aside the tangled set
of Mac OS X bind9 configurations and cross-included zone files, we first have to
find them.

Diving into TimeMachine's backup seems easy enough to start with. The drive has
an actual MBR table, with a HFS+ partition that we are able to mount without any
trouble. Then the directory hierarchy is pretty straight-forward:
`/HostName/Date/HardDriveName/{Content}` and with a bonus on the `Date` layer,
there is a `Latest` symlink!! So far so good, the `Content` layer display the
root of our Mac OS X server filesystem (as it was a month ago)! A big thank you
Apple on that one, we have an ocean of seamless access and easy-to-salvage
backups up to now. etc/dns (that's the directory in which Apple stores the
`bind9` configuration) exists... but it's an empty file (size 0)... and so are
most directories with a depth greater than 2 on the file system. Surely the
content is available somewhere else, but Where‽ After a bit of googling
(<http://hints.macworld.com/article.php?story=20080623213342356>), it appears
that I need to use the inode number of these empty file and go find the content
in the `/.HFS+ Private Directory Data^M/dir_{inode}directory`... Yup, that
folder name got a Carriage Return in it... Good luck with that... Most commands
won't like it and crash in various, colourful and somewhat funny ways. `ln`,
however, is not one of them and symlinking the directory then symlinking its
content to useful names will save us from Oblivion and restore our files!

After a bit more of data-mining and zone files cleansing, we now have a working
DNS server! \[2/3!\] Okay, now it's a bit less problematic: All our users have a
working Internet connection and temporary local accounts have been created on
the locked out machines so everybody is almost able to procrastinate access
Internet and our various customer repositories. The internal services have been
switched to their fallback authentication database when possible and once we've
switched the `resolv.conf` settings on most of our internal servers, the vast
majority of our team is able to go back to work, after only a couple hours of
technical unemployment. Of course, in the meantime, the firewall, gateway and
VPN decided they didn't quite like the new server and we had to make little
adjustments that took almost a day of work to be fine tuned.

We now face the biggest fish of all: The `OpenLDAP`. Sure, the migration plan
was almost well defined, but we still had to find a way to load the old binary
data so we could dump them into an almost-up-to-date LDIF version we could work
with. After a couple of hours of battle with OpenLDAP, turns out you can!
Fortunately, Apple is only using a custom LDAP schema, and not a patched version
of LDAP (Kudos again!). Adjust the apparmor settings to allow a new LDAP db
directory, create your new db and shut down OpenLDAP. Now all we have to do is
dump the OpenLDAP back up from the time machine instead of the new DB, restart
LDAP and use slapcat (which is basically schema agnostic) to extract our data in
LDIF format! We also put the old database as read-only root restricted for
future references. Of course, we can forget about extracting the Kerberos or
ApplePasswordServer passwords hashes but, at least, we have all the data. Since
we are here, we will take the time to clean up the Apple specific attributes and
entries and clean up a few things \[3/3!\]. Oh! And also enable `smbk5pwd` in
our LDAP (there is a little
[PPA](https://launchpad.net/~georg-rath/+archive/ppa-mrce) for ubuntu for
including this module to our LDAP). The last battle was to get the always
over-sensitive Samba to collaborate with the new LDAP server. Now, a mere 4 days
of work after this fateful phone call, we have again everything in working
order, with one obsolete server less and a cleaner setup!
