---
layout: post
title: Migrate Mac OS X 10.6 Open Directory to an Unix OpenLDAP (including passwords!)
---

From what I have seen for the week or so I had to work closely with [our common
best friend](http://www.google.com). Several of us already try this without
success.

Of course, most of you were able to transfer most of the data, but the
passwords... these lousy Mac OS X passwords not stored neither in the LDAP, nor
in any compatible format.

Most of the time, it's quite simple and you will be able to use `slapcat` to
export any data from your Open Directory. Since this is a standard OpenLDAP
migration/clean up process, I won't waste time on it.

How to make your Mac OS X clients to connect using your brand new OpenLDAP is
extensively described in this nice article:
<http://rajeev.name/2006/09/09/integrating-mac-os-x-into-unix-ldap-environment-with-nfs-home-directories/>

This article was actually the key for me. Notice how it's using `authAuthority:
;basic;` to use the OpenLDAP standard userPassword attribute. I thought: that
means this attribute is the key to retrieve my lost passwords.

On a standard Open Directory, you usually have two values for this attribute: one
starting by `;ApplePasswordServer;` and one starting by `;Kerberosv5;`. That
means two different way of retrieving your passwords. However none of these
solutions gives you an hash compatible with OpenLDAP (plain-text, crypt, or
plain/salted MD5/SHA1).

If you're curious, kerberos password hashes can be dumped using `kdb5_util dump
-`, however, I was unable to find a description of what each column means.  The
human readable version (using `kadmin -p {USER}`, `listprincs` and `getprinc`)
does not list any hash compatible with OpenLDAP either.

On the other hand, the Apple Password Server database can be dumped using
`mkpassdb`. Unfortunately, there is still no password hash accepted by OpenLDAP
there. Except the ominous `*cmusaslsecretPPS` for which I have never been able
to find meaning, there are a few more hashes: NT, LM, DIGEST-MD5, CRAM-MD5 and
Kerberos referral (once again)...

Wait a sec... Did someone said NTLM ‽‽ You must be kidding me right? Nope, there
it is just sitting in there waiting for [john](http://www.openwall.com/john/).

Thanks to a [generous
friend](http://osdir.com/ml/macos-x-server/2009-05/msg00422.html), we can easily
extract the list of NTLM hashes (I recommend the LM version, usually digest 2 in
APS). Less than 10 hours later, we got an all caps version of all of our
password. The case is then trivial to check using, for example, `kinit` or `su`.

OK, that's a bit overkill, but nothing is too good for our beloved users isn't
it?
