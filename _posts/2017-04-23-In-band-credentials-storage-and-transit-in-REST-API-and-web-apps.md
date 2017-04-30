---
layout: post
title: "In band credentials storage and transit for REST APIs in the context of Web Apps (Part 2b)"
---

While writing the the second part of
my [Authentication and security for REST API in the context of Web Apps][intro]
series of posts, I realized
that [discussing the different alternatives to store credentials token][part-2]
was more extensive than I previously thought.

This post focuses on storing and transmitting credentials in band. This means
the API is designed in such a way that the underlying Web App is aware and
directly manages the credential token (e.g., Web Storage). The other half of the
discussion focuses on [out-of-band credential storage][part-2a].

Most REST API libraries, frameworks, and essays in general, recommend using
the [Authentication Bearer][http-authentication-bearer] HTTP mechanism. Which is
pretty much perfect from a [REST][rest] point of view, since it
is [stateless][stateless] and handled manually by the app. No browser/user-agent
handle them automatically the way they handle cookies, yet.

In any case, even if the tokens need to be manually managed by the app, it
wouldn't be practical to require the developer to do it independently for each
and every request. The credentials and authentication management part MUST be
segregated from the business logic part of the application's code. Both to avoid
code duplication, which is error-prone and hard to maintain, and to reduce the
exposition surface of the tokens. Having a smaller amount of code dealing with
credentials and tokens will mean requiring an attacker to analyze more code
before being able to exploit any potential [XSS][xss] vulnerabilities to steal
the token. Once this segregation is in place, the tokens would basically become
an out-of-band resource from the point of view of the business logic part of the
app.

Either way, an attacker could trivially either duplicate the authentication code
or leverage the authentication library of the attacked app. Therefore, none of
these solutions will prevent [XSS vulnerabilities][xss] in any ways and they
MUST be addressed in other ways.

Since the token is available in-band to the Web App itself, an attacker would
not need to trigger [XST][xst] or other convoluted attack to be able to steal the
token.

[REST][rest] and [HATEOAS][hateoas]'s goal is to make machine crawling of an API
easy. They aim to be exploitable by software with no _a priori_ knowledge of the
API or the service behind it. Therefore, an attacker does not need to steal or
copy any token. Simply by leveraging the API discoverability feature itself, they
will be able to extract every single piece of information from the API,
automatically.

It is important to note, that Web storage APIs,
although [broadly implemented and available][caniuse:web-storage], still have
some issues or unavailability depending on the browser configuration
(e.g. private browsing).

For these reasons, some Web App and API will still require cookie-based token
management. This can be mitigating by restraining the access to this feature in
the API and forcing tokens to include how they should be used (as a Bearer
manually managed token or via cookies).

There are several ways to store tokens in-band. Each with their pros and cons.

### An encapsulated JavaScript variable

The biggest advantage to this solution is technology availability. Any browser
able to handle an [SPA][spa] will be able to store a variable somewhere and send
it back to the server.

On the cons side, this solution offers no permanent storage (no long term tokens
can be effectively used). It will also cut the user from well known and expected
behaviors, like session forking (opening a link in a new tab or window, while
keeping the user's session active).

By opening a link in a separate tab, the user will load a new instance of the
Web App, which doesn't have any access to the JavaScript run time environment of
the previous app. This will require the user to login again.

In cases where the App is designed in such a way that this this behavior would
create issues anyway, this might be an acceptable solution for short term
tokens. However, No design choice or workflow example fitting this description
would still be provided by a [stateless][stateless] Web App and API.

This solution SHOULD NOT be used unless the browser environment is such that
session forking is impossible.

### [Session storage][session-storage]

Session storage is to local storage, what session cookies are to
permanent/time-based expiration cookies.

Session storage is basically a local storage bound to a session. This is not a
permanent storage. Therefore, this is not appropriate for long term tokens.

However, session storage specification requires supporting session
forking. There are still some use cases the user wouldn't get their session
storage back (test for the feature is available
at <http://jsfiddle.net/moucdygg/2/>).

Future implementation will possibly include more use cases in what is considered
session forking. Thus, making session storage more and more reliable as a
acceptable short-term token storage. Moreover, using a "remember me" token in a
permanent storage (even a short term one), could mitigate these issues quite
effectively.

Plus, since the storage is automatically cleared when the session is closed,
this will reduce potentially higher value (explicitly authenticated) short term
tokens' exposure, even if just by a few minutes.

By creating a new browser tab or window from scratch, a user would rely on a
clear session storage. An App relying on session storage for short term tokens
would provide a better experience to users trying to manage multiple accounts at
once.

When possible, session storage SHOULD be used as a short term token store.

### [Credential Management][cm-api]

The credential management specification is designed to allow browsers and
user-agents to easily and securely store and manage user
credentials. Credentials are provided to the Web App either explicitly (with the
user authorization) or silently (without the user knowledge). They are not
intended to be harder to use than standard Web storage, but offer better user
control.

Short term tokens MAY be stored in Credential Management storage, but it offers
no discernible advantage at the moment.

However, credentials are segregated by origin, but not by session. Thus, if
allowing the user to open multiple sessions at once (e.g., by creating multiple
tabs or window not sharing session storage) is a desirable feature, Credential
Management storage will most probably not be able to store short-term tokens in
this context.

On the other hand, Credential Management storage is designed and extremely
well-suited to host long-term tokens. It SHOULD be used to store
these. Unfortunately, at the time of writing this, Credential Management API
is [far from being widely supported][caniuse:cm-api]. Other solutions will
therefore be required until it becomes widely available.

Whether to use it or not for user credentials is outside the scope of the
current discussion.

### [Local storage][local-storage]

Local storage is, currently, the only widely supported manually managed
permanent storage available to Web Apps. Local storage is safely segregated
using the same-origin policy and offer cross-session durability.

However, given their durability, and the fact that there is no way to expire
data in it, explicitly authenticated (high value) tokens SHOULD NOT be stored in
local storage. If the support for session forking of the session storage is
deemed inappropriate short term tokens MAY be stored in local storage. However,
in general, short term token are more appropriately stored in session storage
and SHOULD NOT be stored in local storage.

Long term tokens are more appropriately stored in secured credentials specific
storage (such as [Credential Management][cm-api] storage). However, until the
implementation of such credential-specific secure storage is widely available,
local storage MAY be used to store long term tokens.

In the next part, we will focus on designing a [RESTful API to manage these
tokens][part-3].

[intro]:   /2017/04/22/REST-APIs-authentication-and-security.html                            "Authentication and security for REST API in the context of Web Apps"
[part-2]:  /2017/04/23/Credentials-storage-and-transit-in-REST-API-and-web-apps.html         "Credentials storage and transit for REST APIs in the context of Web Apps"
[part-2a]: /2017/04/23/Out-of-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html "Out-of-band credentials storage and transit for REST APIs"

[rest]:      https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm "Architectural Styles and the Design of Network-based Software Architectures"
[stateless]: https://en.wikipedia.org/wiki/Stateless_protocol            "Stateless protocol"
[hateoas]:   https://en.wikipedia.org/wiki/HATEOAS                       "Hypermedia As The Engine Of Application State"
[spa]:       https://en.wikipedia.org/wiki/Single-page_application        "Single-page application"

[xss]: https://www.owasp.org/index.php/Cross-site_Scripting_(XSS) "Cross-site Scripting (XSS) "
[xst]: https://www.owasp.org/index.php/Cross_Site_Tracing         "Cross Site Tracing"

[http-authentication-bearer]: https://tools.ietf.org/html/rfc6750#section-2.1                "Authorization Framework: Bearer Token Usage - Authorization Request Header Field"
[local-storage]:              https://www.w3.org/TR/webstorage/                              "Web Storage (Second Edition)"
[session-storage]:            https://www.w3.org/TR/webstorage/#the-sessionstorage-attribute "Web Storage (Second Edition) - The sessionStorage attribute"
[cm-api]:                     https://w3c.github.io/webappsec-credential-management          "Credential Management Level 1"

[caniuse:web-storage]: http://caniuse.com/#feat=namevalue-storage     "Can I use? - Web Storage"
[caniuse:cm-api]:      http://caniuse.com/#feat=credential-management "Can I use? - Credential Management API"

