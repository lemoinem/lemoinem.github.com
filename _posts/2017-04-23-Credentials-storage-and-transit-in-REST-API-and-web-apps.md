---
layout: post
title: "Credentials storage and transit for REST APIs in the context of Web Apps (Part 2)"
---

This is the second part of my [Authentatication and security for REST API in the
context of Web Apps](/2017/04/22/REST-APIs-authentication-and-security.html)
serie of posts. Now, that we discussed how to design your credentials token, we will
try to find the best way to store them.

Having secure token that limit the user and service's exposure if they ever are leaked
is a good think. However, having a Web App or API leaking credentials is a terrible awful
thing. One can never be sure every leaks and holes have been addressed and this is why
security issues should be mitigated in several ways rather than with a single faillible
silver bullet. A Single Point of Faillure is just as bad whether it involved availability
or security (Some may argue is even worst in the case of security).

So, how should we store our tokens?

Within modern browsers, there are up to 6 ways to store tokens, split in two categories:
* [Out-of-band](/2017/04/23/Out-of-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html)
  (the app doesn't have access to the token, it's managed automatically by the browser)
  * Manually managed out-of-band credential stores (for example, inspired from the [Credentials Management API](https://w3c.github.io/webappsec-credential-management))
  * Cookies (3)
* [In-band](/2017/04/23/In-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html)
  * A JavaScript variable (preferably thoroughly encapsulated so it cannot be accessed directly)
  * [Session storage](https://www.w3.org/TR/webstorage/#the-sessionstorage-attribute)
  * [Credential management](https://w3c.github.io/webappsec-credential-management)
  * [Local storage](https://www.w3.org/TR/webstorage/)

Since each solution and approach has many points to be discussed, each category
is discussed separately in their own blog posts.

Here, only a summary of the strongest suggestions is provided.

Cookies SHOULD NOT be used to store REST authentication tokens. However,
technological limitations will require us to provide an as RESTful as possible
way to use them.

Session storage SHOULD be used as a short term token store.

Credential Management storage SHOULD be used to store long term tokens. However, technological
limitations will require us to fall back to local storage for the time being.
