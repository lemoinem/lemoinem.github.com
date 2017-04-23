---
layout: post
title: "Credentials format for REST APIs in the context of Web Apps (Part 1)"
---

This is the first part of my [Authentatication and security for REST API in the
context of Web Apps](/2017/04/22/REST-APIs-authentication-and-security.html)
serie of posts. In this one, we will focus on how to design credentials
containers (token and such).

When one says authentication, one automatically thinks about credentials.

Up to now, the basic way to store credentials in a Web App, was through the use of sessions:
You require the user to send an opaque session ID to your server with each and every request.
The server was storing information such as who the user is and what is their last CSRF token,
and what is their current language and so one.

Unfortunately, this is not very RESTful (1), REST APIs and Apps should be stateless.
A session is most definitely stateful.
Second, this is not very scalable, since all the servers need to have access to the same
session repository, this would imply storing sessions in database or other solutions which
are complex and error-prone.

In the context of JSON-LD (2), JWT are definitely the way to go. However, a simple JWS
(as is the current practice) would allow any attacking party to identify the user
by using the stored userID or other identifying mean (8).

A JWE, including both a tamper-proof publicly readable part, and an encrypted part is the way to go.

If we want the token to be truly self-containing, additional information will be required
to be stored in it. Such as the access-level of the user (or Roles), UserGroups it belongs
to and so on.

In the context of a Web App, these are data that could be usefull to the front-end SPA. However, a
better way to provide this information to the front-end, would be by using HATEOAS (1) and API
discoverability (more on this later).

### How to design your REST Credentials Container or Token (JWT)

A JWT with the JWE format MUST be used. The data included in the public part MUST include:
* `exp`/Expiration date (the client needs to know when to renew their credentials)

The data included in the encrypted part MUST include:
* `aud`/Audience (Origin scope of the token, used for CSRF protection,
  MAY be skipped or set to null in some cases, 4)
* `sub`/Subject (userID)
* In case of an app/API supporting external authentication: `iss`/Issuer
* `jti`/JWT ID: a uniq ID, used as a nonce to prevent easy identification and tracking
  of the `sub` claim (8). Without a nonce, the encrypted part would be constant for a given
  user.
* `iat`/Issued at, `nbf`/Not Before, or a boolean flag specifying the
  authentication level (6). If iat or nbf is used, then the validity interval
  of the token can be used to identify the token as short term (explicit authentication)
  or long term (remembered).

Short term tokens SHOULD have a lifetime of a few hours at most:
* SHOULD be more than: 30mins
* recommended value: 1h
* MUST be less than 4h

Long term tokens SHOULD have a lifetime of a few days or weeks:
* SHOULD BE more than: 1wk
* recommended value: 1mth
* MUST be less than 1yr

Short term tokens SHOULD be renewed automatically (but explicitly,
Resources shouldn't be side-updated and tokens are a resource)
by the app when a user interaction occurs after the token has reached half its lifetime.
Token SHOULD be renewed only when the user interaction requires a communication with
the server, unless the token is on the brink of expiration (a few minutes away).
If a User stays inactive on the front-end App for a time longer than the token lifetime,
the token SHOULD NOT be renewed and the user SHOULD be asked to be reauthenticated.

A short term token MUST NOT be renewed into a long term token without an explicit request from
the user to do so and this _request_ (not only the token itself) SHOULD be explicitly authenticated.
An explicitly authenticated request for a long term token prevents both CSRF and XSS attacks leveraging
short term tokens to produce long term tokens. This will force the attacker to refresh the stolen
token on a regular basis, thus increasing the resources they need to impersonate a broad number of
users.

An explicitly authenticated short term token SHOULD be renewed in a remembered short term token (i.e. the level
of authentication is allowed to be dropped automatically). This will prevent users to stay explicitly authenticated
for a long time. If any attacker was able to steal (e.g. through XSS) an explicitly authenticated token,
they would have a limited time to exploit this token to its full potential.

A long term/remembered token produced by an explicitly authenticated request MAY be used to
represent an explicit authentication for a duration equivalent to the lifetime of a short term token
requested in the same way (minus the long-term requirement) and at the same time.

A long term token SHOULD be used to generate a remembered short term token at the begining of a session.
This will reduce the exposition of any long term tokens and will prevent prevent their stealing.

Long term token MAY be renewed automatically once they reached half their lifetime, like short term tokens.
The application SHOULD get confirmation from the user to renew them and SHOULD require explicit authentication
to generate any long-term token.

Additional data such as a user's Roles or Groups SHOULD be stored in the encrypted part, to
prevent correlating these informations in order to track the user.
