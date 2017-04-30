---
layout: post
title: "Credentials storage and transit for REST APIs in the context of Web Apps (Part 2)"
---

This is the second part of
my [Authentication and security for REST API in the context of Web Apps][intro]
series of posts. In the previous post, we designed
a [strong credentials' vessel (token)][part-1]. In this post, we will try to
find the best way to store and provide them to the API.

Having secure tokens that limit the user and service's exposure if they are ever
leaked is a good thing. However, having a Web App or API leaking credentials is
a terribly awful thing. One can never be sure every leaks and holes have been
addressed. This is why security issues should be mitigated in several ways and
layers rather than with a single fallible pseudo-silver bullet. A Single Point
of Failure is just as bad whether it involved availability or security (Some may
argue it is even worst in the context of security).

So, how should we store our tokens?

Modern browsers, support up to 6 ways to store tokens. For easier
considerations, we will split these in two categories:
* [Out-of-band][part-2a], the app doesn't have access to the token, it's
  provided to the server automatically by the user-agent:
  * Manually managed out-of-band credential stores (for example, inspired from
    the [Credentials Management API][cm-api]),
  * Cookies (criteria 3);
* [In-band][part-2b]:
  * as a JavaScript variable (preferably thoroughly encapsulated so it cannot be
    accessed directly),
  * in the [session storage][session-storage],
  * using the [Credential Management API][cm-api],
  * in the [local storage][local-storage].

Since each solution and approach has many points to be discussed, each category
is discussed separately in their own blog posts
([out-of-band post][part-2a], [in-band post][part-2b])..

Here, only a summary of the strongest suggestions is provided.

Cookies SHOULD NOT be used to store RESTful authentication tokens. However,
technological limitations might require us to provide them to legacy or specific
user-agents.

Session storage SHOULD be used as a short term token store.

Credential Management storage SHOULD be used to store long term tokens. However,
technological limitations might require us to rely on local storage instead.
being.

In the next part, we will focus on designing a [RESTful API to manage these
tokens][part-3].

[intro]:   /2017/04/22/REST-APIs-authentication-and-security.html                                "Authentication and security for REST API in the context of Web Apps"
[part-1]:  /2017/04/23/credentials-for-REST-API.html                                             "Token's format (Credentials)"
[part-2a]: /2017/04/23/Out-of-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html "Out-of-band credentials storage and transit for REST APIs"
[part-2b]: /2017/04/23/In-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html     "In band credentials storage and transit for REST APIs"
[part-3]:  /2017/04/23/REST-authentication-API-for-Web-App.html                                  "RESTful authentication API"

[local-storage]:   https://www.w3.org/TR/webstorage/                              "Web Storage (Second Edition)"
[session-storage]: https://www.w3.org/TR/webstorage/#the-sessionstorage-attribute "Web Storage (Second Edition) - The sessionStorage attribute"
[cm-api]:          https://w3c.github.io/webappsec-credential-management          "Credential Management Level 1"
