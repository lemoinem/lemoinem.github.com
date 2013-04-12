---
layout: post
title: "Centralizing servers access"
---

\[Disclaimer: Here is a blog-post I wrote for my company blog. You can find the
original text at <http://www.mentel.com/centralizing-servers-access/>. They
kindly allow me to publish a copy on here!\]

Managing a server farm and keeping all accounts, passwords and ssh keys up to
date and synchronized can be Hell. One of the solutions for that is to use
[Puppet](https://puppetlabs.com/) or
[Chef](http://wiki.opscode.com/display/chef/Home). But these mean active
syncing, and if your users database starts to grow bigger than something mildly
useful, you're going to have bigger worries and better use than keeping access
up-to-day.

Another reason why I think active maintenance is not the right way to maintain
access up-to-date is, compared to whatever other records that will be managed
that way, they are mostly stable. Your sysadmin (or even devs) team won't be
changing that often, and even the most paranoid sysadmins won't change their
password more than, say, every month or couple of months. Even if the team grows
to a medium-to-big directory, it will mean you need to manage a consequent set
of projects, many starting and delivered at a much faster pace than your
sysadmin team changes.

Thus, what you need is a pull based central system. After years of looking for
the right one, I think I just did. LDAP has been for a long time the wannabe
solution to this problem. But there is a major drawback: What if your OpenLDAP
server crashes? or what if your server simply loses connectivity to it? That's
it, you're locked out of your server. And most of the time, it's going to be at
the worst time ever... Remember: Murphy's Law! Moreover, there is no current way
to manage ssh public keys via LDAP. The only way to do it, would be to use an
annex file updated via a cronjob, listing of the users and retrieving and
formatting the keys for each user. It's a hassle and similar to active
maintenance in a way I'm not so fond of.

To solve this problem, we therefore need a local middle-man with an OpenSSH
support. [nscd](http://linux.die.net/man/8/nscd) could be a solution, but there
is no OpenSSH support and the cache has to be manually cleared on a regular
basis to avoid out-of-date record (as simple as a crobjob, but still a
hassle). [SSSD](https://fedorahosted.org/sssd/) is the one. The cache is easy
enough to configure, works like a charm and can be configured so it's trusted
only when the LDAP server is out-of-reach.

The SSSD project being run by the Fedora/Red Hat team, the configuration process
follow a very Unix way of doing things: One step at a time. The first step is
configuring the ldap-tools and making sure your new server can reach the LDAP
and fetch all the information it needs just fine. That shouldn't be more than 4
config lines in
[`ldap.conf`](https://gist.github.com/lemoinem/5208310#file-ldap-conf). Then
configure `sssd` to use it as a backend. The config is a bit lengthier ([25
lines](https://gist.github.com/lemoinem/5208310#file-sssd-conf)) but it includes
everything you'll ever need from SSSD. Important configs here are `enumerate =
true`, `services` and the LDAP domain config. `enumerate = true` is actually
advised against unless you have a real reason to use it. In my case the servers
are often left unattended for a very long period of time or at least deployers
are going to connect to server using application specific login (each of my
websites has its own user that's used for deploying). This means sysadmins
account will probably not be pre-populated. Make sure this is the case for you
too since using enumeration could induce delays and high load on the LDAP
server. I still recommand to use `enumerate = true` to start with so you can
make sure the nss bridge works fine, but remember to disable it afterward if you
have no use for it. You probably should leave `cache_credential` to `false` to
start with too.

As far as I know, there is no direct way to query the SSSD daemon. The simplest
way to test the config is to add SSSD as a nss responder. Modify
[`nsswitch.conf`](https://gist.github.com/lemoinem/5208310#file-nsswitch-conf-extract)
to use SSSD for `passwd`, `group` and `shadow` (eventually netgroup too). You
should then be able to see your users and groups via getent passwd and getent
group. However, unless you're using a different config for SSSD allowing it to
see your password, the users shouldn't show up in `getent shadow`.

Second part: make authentication work. For most reasonably recent Linux/Unix
distributions, PAM is used for authentication. Configuring PAM to use sss for
authentication is quite easy, only four lines in the pam.d directory (see
<https://gist.github.com/lemoinem/5208310#file-pam-d-summary>), focus on the
`pam_sss` lines, order is important). If you changed the `sssd.conf`, make sure
the pam section is there and it's listed in services. Once this is done, you
should be able to login with your LDAP users. And if you have trouble with home
directories not existing, take a look at the `pam_mkhomedir.so` line. Just like
that!

Next step, I'd recommend to enable caching. Important configs are
`cache_credentials`, `offline_credentials_expirations` and
`entry_cache_timeout`. The first one is speaking for itself. Of the other two,
the former (when set to 0) disables cache expiration, so the credentials will be
kept as long as needed. The later one (when set to 1), means that each entry
fetched from the LDAP should be refreshed if they're older than 1
minute. Bottom-line: data is stored indefinitely, but refreshed every so often:
exactly the behavior we were looking for! Testing caching could be done two
ways: First disable networking on the server entirely (If you can access your
server some other way around). Second, rewrite the LDAP hostname to another or
even a non-existent IP (Using `/etc/hosts`).

So, now we have everything we need authentication-wise. What about SSH? Well,
you may have notice three lines mentioning ssh in my `sssd.conf`. Just adding
these will allow SSSD to provide ssh key support. You need a fairly recent
version of SSSD however, 1.9+. To store the public keys in your users, you need
to use this LDAP schema:
<https://gist.github.com/lemoinem/5208310#file-openssh_user_public_key-schema-ldiff>. The
keys themselves can be stored just the way you would do it in an `authorized_keys`
file, no restrictions.

Now, you can be sure that having such a clean API as SSSD, it won't be walking
around, creating or updating `authorized_keys` files around... There is a
command-line tool, something part of sssd-tools, called
`sss_ssh_authorized`. And it's doing exactly what you probably think. Give it a
username, it will retrieve the ssh keys associated to it. The ones SSSD knows
about anyway. You can call it directly to test if SSSD is recognizing your ssh
keys the right way. But the problem is now how to feed that to OpenSSH... Looks
like we are back to the good old cronjob solution? NOT! Of course...

You will be really interested to read about this patch, that just made it
upstream into OpenSSH (released as part of the brand new 6.2):
<https://bugzilla.mindrot.org/show_bug.cgi?id=1663>. It's creating two new
configuration keywords for sshd: `AuthorizedKeysCommand` and
`AuthorizedKeysCommandRunAs`. They are doing exactly what they says: calls a
command-line script (with a username as parameter) and expects it to reply with
a list of public keys. And there you go, the loop is closed! (And about time
too)...

I've made a couple of tests and had everything running smoothly for about two
weeks now, and while I haven't thoroughly tested it, I encourage everybody to
test it and use it! It seems stable enough to be set up into production (with a
fallback local sudoer user). Of course, most of current PROD approved
distributions won't be included these fairly recent patches or versions of SSSD
and OpenSSH. If you're using Ubuntu however, you're in for a treat, since there
is a PPA for that so it can installed on the last LTS (12.04, Precise):
<https://launchpad.net/~nicholas-hatch/+archive/auth>.

PS: Of course, you don't always want everybody to be able to login into your
servers, but that's what [pam_access](http://linux.die.net/man/5/access.conf) is
for.

PPS: If you want to help improve SSSD and its tool suite, you may want to take a
look to [F19 test
days](http://fedoraproject.org/wiki/QA/Fedora_19_test_days). There are three
related test days currently planed: 2013-05-09 for SSSD Improve and AD
Integration, 2013-04-18 and 2013-06-06 for FreeIPA (AD and MFA).
