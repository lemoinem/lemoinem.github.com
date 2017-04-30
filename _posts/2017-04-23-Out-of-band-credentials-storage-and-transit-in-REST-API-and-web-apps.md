---
layout: post
title: "Out-of-band credentials storage and transit for REST APIs in the context of Web Apps (Part 2a)"
---

While writing the the second part of
my [Authentication and security for REST API in the context of Web Apps][intro]
series of posts, I realized
that [discussing the different alternatives to store credentials token][part-2]
was more extensive than I previously thought.

This post focuses on storing and transmitting credentials out-of-band. This
means the API is designed in such a way that the user-agent should not have
access to the credential token directly (e.g., cookies). The other half of the
discussion focuses on [in band credential storage][part-2b].

If you haven't read Roy T. Fielding's dissertation yet, you should at least go
check his section on "[Why are cookies not RESTful?][rest-cookies]". It is a
much better explanation than I could ever write up. His arguments basically
applies to any automatically managed out-of-band storage.

Moreover, once the authentication token is completely out or reach of your app,
you need an alternative API so your App still knows when to renew your token and
have a way to access the additional data required to present a better UX (User
roles and such).

### Manually managed, out-of-band credential stores.

Since out-of-band credential stores present challenges for Web App UX, their use
cases are mostly limited to manage user credentials. Long term "Remember me"
tokens can be construed as a form of user credentials. Any client with this
token will be allowed to open a session under the identity of this specific
user. Moreover, we already recommended to trade-in such long term tokens for a
short term one as early as possible during a session. This is very similar to
how user credentials are processed.

There are, currently, no manually managed out-of-band storage and transit
solution.  The [Credentials Management API][cm-api] could fit this concept.
However, the current specification draft requires the Web App to be able to
access the private details of the stored credentials and manage how they are
used to interact with the server. This makes it an in-band storage and is
discussed in a different post.

### Cookies

Given that cookies have been the preferred way to store session IDs for decades
now, many security features have been included to
them. [HttpOnly and Secure][secure-cookies] cookies are completely shielded from
normal user code. This would prevent completely an [XSS][xss] attack from
stealing and storing the token for later use. However, there are still other
attack vectors (known and unknown), such as [XST][xst], that could bypass these
protections.

There is also the following point to consider: [REST][rest]
and [HATEOAS][hateoas]'s goal is to make machine crawling of an API easy. They
aim to be exploitable by software with no _a priori_ knowledge of the API or the
service behind it. Therefore, an attacker does not need to steal or copy any
token. Simply by leveraging the API discoverability feature itself, they will be
able to extract every single piece of information from the API, automatically.

Finally, having the browser automatically manage the tokens gives rise to the
temptation to have the server seamlessly renew tokens automatically. This
creates two additional issues:

First, since each request can carry two resources (a new token and whatever the
result of the request is), it is not RESTful. Plus you cannot cache anything
anymore. A, potentially expired, token might be served to the client, which is
an issue even for local, private caches (such as a browser's own cache).

The second issue has to do with the long-term/short-term tokens separation. In
our initial suggestions for [credentials container][part-1], we recommend
long-term tokens should be, basically, treated as user credentials. Only ever to
be used to generate a short-term token. The goal of this segregation is to
prevent long-term tokens lingering exposed on the wire and manipulated
continuously either by the app or the browser (thus increasing attack surface
and theft possibility).

Since cookies cannot be stored off the wire by a browser, swapping a long term
token for a short term one can only be done in one of three ways:
1. Both cookies have different names and are used side-by-side, this doesn't
reduce exposure of the long-term token as it is still sent with every request.
2. The short-term token overwrites the long-term one, the long-term token is
lost and replaced by the short-term one. Once the short-term token is expired,
the browser will not be able to bring the long-term token back. This will
effectively preclude any use of long term tokens.
3. Override the long-term token with a short term-token embedding the long-term
one. In order to keep the benefits of the long-term support token, the cookie
will need to have a lifetime equivalent to the long-term token. In the case
where the short term token is expired, the server must allow the short term
token to be renewed into another new short term token by using the embedded long
term token. In effect, the API is just manipulating bigger and more complex long
term tokens.

Any out-of-band data automatically managed by the browser is highly vulnerable
to both types of [CSRF][csrf] attacks (impersonation and cookie injection),
cookies included.

Non-browser agents can be classified in two categories when considering cookies:

1. Proxies (e.g., caches) will mostly be disabled or confused by cookies. A
   cached response including cookies will potentially trigger unwilling
   impersonation or change the end-user client's state in a way not intended by
   the server.
2. Native applications or other end-user clients will be responsible to process
   the whole HTTP Request/Response round-trip. In this case, cookies are not
   out-of-band data anymore and must be stored and managed explicitly by the
   client. In this context, tokens become a real in-band resource, it should
   therefore be manipulated just like any other resources offered by the
   API. Cookies are usually not the preferred way to send resources
   representations for HTTP REST APIs.

For all these reasons, cookies SHOULD NOT be used to store REST authentication
tokens.

However, there are case where the combination of software environment and UX
requirements is such that cookies are the only option available... For this
reason and this reason only, an integration of cookie-based tokens will be
discussed in further parts of this series.

The other half of the discussion focuses
on [in band credential storage][part-2b].

[intro]:   /2017/04/22/REST-APIs-authentication-and-security.html                            "Authentication and security for REST API in the context of Web Apps"
[part-1]:  /2017/04/23/credentials-for-REST-API.html                                         "Token's format (Credentials)"
[part-2]:  /2017/04/23/Credentials-storage-and-transit-in-REST-API-and-web-apps.html         "Credentials storage and transit for REST APIs in the context of Web Apps"
[part-2b]: /2017/04/23/In-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html "In band credentials storage and transit for REST APIs"

[rest]:      https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm "Architectural Styles and the Design of Network-based Software Architectures"
[hateoas]:   https://en.wikipedia.org/wiki/HATEOAS                       "Hypermedia As The Engine Of Application State"

[rest-cookies]:   https://www.ics.uci.edu/~fielding/pubs/dissertation/evaluation.htm#sec_6_3_4_2
[secure-cookies]: http://resources.infosecinstitute.com/securing-cookies-httponly-secure-flags/

[csrf]: https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF) "Cross-Site Request Forgery (CSRF)"
[xss]:  https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)        "Cross-site Scripting (XSS) "
[xst]:  https://www.owasp.org/index.php/Cross_Site_Tracing                "Cross Site Tracing"

[cm-api]: https://w3c.github.io/webappsec-credential-management "Credential Management Level 1"
