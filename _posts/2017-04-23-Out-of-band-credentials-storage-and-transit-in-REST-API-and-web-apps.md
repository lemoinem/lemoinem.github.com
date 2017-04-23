---
layout: post
title: "Out-of-band credentials storage and transit for REST APIs in the context of Web Apps (Part 2a)"
---

While writting the the second part of my [Authentatication and security for REST API in the
context of Web Apps](/2017/04/22/REST-APIs-authentication-and-security.html)
serie of posts, I discovered that [discussing the different alternatives to store credentials token](/2017/04/23/Credentials-storage-and-transit-in-REST-API-and-web-apps.html)
was more extensive than I previously thought.

This post focuses on storing and transmitting credentials out-of-band. This means the
API is designed in such a way that the underlying Web App should not does not have access
to the credential token directly (e.g., cookies). The other half of the discussion focuses on
[in band credential storage](/2017/04/23/In-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html).

If you haven't read Roy T. Fielding's dissertation yet, you should at least go check to his paragraph
on "[Why cookies are not RESTful?](https://www.ics.uci.edu/~fielding/pubs/dissertation/evaluation.htm#sec_6_3_4_2)".
It's a much better explanation than I could ever come with.

This would basically apply to any automatically managed out-of-band storage.

Moreover, once the authentication token is completely out or reach of your app, you need an alternative
API so your App still knows when to renew your token and have a way to access the additional data required to
present a better UX (User roles and such).

#### Manually managed, out-of-band credential stores.

Since out-of-band credential stores present challenges for Web App UX, their use cases are mostly limited to
manage "Remember me" tokens/credentials or primary credentials. Another solution would be to
craft the API in such a way that UX related meta-data (the public part of the token) is made available
to the app.

There are, currently, no manually managed out-of-band storage and transit solution.
The [Credentials Management API](https://w3c.github.io/webappsec-credential-management) could fit this concept.
However, the current specification draft requires the Web App to be able to access the private details of the stored
credentials and manage how they are used to interact with the server. This is an in-band storage and is discussed in
a differnt post.

#### Cookies

Given that cookies have been the prefered way to store session IDs for years
and decades now, many security features have been included to them.
[HttpOnly and Secure](http://resources.infosecinstitute.com/securing-cookies-httponly-secure-flags/)
cookies are completely shielded from normal user code. This would prevent an XSS attack from stealing
and storing the cookie for later use. Even then, there are still other attack vectors (known and unknown),
such as XST, that could bypass these protections.

There is also the following point to consider: REST and HATEOAS's goal is to make machine crawling of an
API easy, using only software without a priori knowledge of the API or service behind it.
Therefore an attacker does not need to steal and copy any token. Simply by leveraing
your own Web App's library, they will be able to extract every single piece of information
from your API, automatically.

Having the browser automatically manage the tokens gives rise to the temptation to have the server
seamlessly renew tokens automatically. This creates two additional issues:

First, since each request can carry two resources (a new token and whatever
the result of the request is), it's not RESTful. Plus you cannot cache anything anymore.
Otherwise a, potentially expired, token might be served to the browser, which is an issue
even for local, private caches (such as a browser's cache).

The second issue has to do with the long-term/short-term tokens separation. In my suggestions for
credential container, I recommend long-term tokens should, basically, only ever be used to generate
a short-term token. The goal of this is to prevent long-term tokens lingering exposed
on the wire and manipulated continuously either by the app or the browser (thus increasing attack surface).

Since cookies cannot be stored off the wire, swapping a long term token for a short term one can only be done
in one of three ways:
1. Both cookies have different names and are used side-by-side,
this doesn't reduce exposure of the long-term token as it is still sent with every request.
2. The short-term token overwrite the long-term one, the long-term token is lost and replaced by
the short-term token. Once the short-term token is expired, the browser will not be able to bring the long-term token back.
This is basically useless.
3. Override the long-term token with a short term-token embedding the long-term one. In order to keep the benefits of the
long-term support token, the cookie will need to have a lifetime equivalent to the long-term token. Which mean, even if your
short term token is expired, it must still be allowed to be used, seamlessly, to be traded into a new short term token,
just like a standard long term token. In effect, you will just be sending a bigger and more complex long term token.

Any out-of-band data automatically managed by the browser is highly vulnerable to
both types of CSRF attacks (impersonation and cookie injection), cookies included.

Non-browser agents can be classified in two categories when considering cookies:

1. Proxies (mostly caches) will mostly be disabled or confused by cookies. A cached response including cookies will potentially
   trigger unwilling impersonation of change the end-user client's state in a way not intended by the server.
2. API or other end-user clients will be responsible to processing the whole HTTP Request/Response round-trip. In this case,
   cookies are not out-of-band data anymore and must be stored and managed explictly by the client.
   In this context, tokens become a real in-band resource, it should therefore be manipulated just like any other resources
   offered by the API. Cookies are usually not the prefered way to send resources representations for HTTP REST APIs.

For all these reasons, cookies SHOULD NOT be used to store REST authentication tokens.

However, sometimes, the combination of software environment and UX requirements is such
that cookies are the only option available...
For this sole reason, an integration of cookie based tokens will be provided in further discussion.
