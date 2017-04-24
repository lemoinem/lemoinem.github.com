---
layout: post
title: "Authentatication and security for REST API in the context of Web Apps (Intro)"
---

I am, finally, starting to look at developing REST APIs as the backend of a Web
Apps.  I'm certainly no the first and we are starting to have great tools and
technologies for this.

In particular, given my background with PHP ans Symfony, I've been looking at:
* [HATEOAS](https://en.wikipedia.org/wiki/HATEOAS) (that name, though...),
* [JSON-LD](http://json-ld.org/),
* [JWT](https://jwt.io/),
* [HYDRA](http://www.markus-lanthaler.com/hydra/),
* [API Platform](https://api-platform.com/)

For anyone, who haven't read [Roy T. Fielding's
thesis](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm), I would
highly recommend it. As much because it is THE document birthing the REST
architectural style, as because it offers extremely interesting insight on Web
and HTTP technologies.

However, there is a subject on which I haven't found much resources, which is
"How to design a RESTful and secure authentication API?" and the topic that
obviously follow it: "How to integrate this with a RESTful API".  Most platforms
and tools focus on the API itself, putting security aside as a secondary
concern. Even API Platform, which is a great platform, extremely developer
friendly, and with an active community, doesn't even try to offer [decent
default security
settings](https://github.com/api-platform/api-platform/issues/109) yet.

With the following posts, I will try to provide my thoughts and ideas on the
subject.

My main concerns are (in no particular order, but numbered for easy reference):
1. As REST as possible, including HATEOAS
2. Based on JSON-LD and Hydra (but should support or be adapted to other media
   types easily/seamlessly)
3. Since we are talking about Web Apps, Cookies will still be discussed
4. Taking care of CSRF Vulnerabilities (including [Login
   CSRF](http://www.adambarth.com/papers/2008/barth-jackson-mitchell-b.pdf))
5. XSS vulnerabilities, while out-of-scope (beacause it's a fatal flaw for an
   SPA), should be mitigated when possible.
6. Should support two level auth (remembered & explicitly authenticated)
7. Should support password-based, token-based, MFA, or external trustee
   (OAuth/OpenID) authentication easily
8. If a token is ever leaked, a third party shouldn’t be able to link it easily
   to a used (remove user-tracking capabilities)
9. Should limit account brute-forcing
10. Should limit or mitigate phishing attacks if possible.

I will try to address and discuss these concerns and offer better insight on
what should be important and why in the following posts:

1. [Credentials (tokens)](/2017/04/23/credentials-for-REST-API.html)
2. [Credentials storage and transit](2017/04/23/Credentials-storage-and-transit-in-REST-API-and-web-apps.html)
    1. [Out-of-band credentials storage and transit](2017/04/23/Out-of-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html)
    2. [In band credentials storage and transit](2017/04/23/In-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html)
