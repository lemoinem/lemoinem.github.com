---
layout: post
title: "Credentials format for REST APIs in the context of Web Apps (Part 1)"
---

This is the first part of
my
[Authentication and security for REST API in the context of Web Apps][intro]
series of posts. In this one, we will focus on how to design credentials
containers (tokens and such).

When one says authentication, one automatically thinks about credentials.

In the context of a Web App, there are two kinds of credentials:
* User credentials: which are used to prove a user is actually who they claim
  to be,
* Request credentials: Which are used to authenticate a request as being issued
  by a specific user, and then answer the request based on what this user is
  allowed to access on the platform.

User credentials examples include:
* a pair username/password,
* a correct response to a MFA challenge,
* a authentication certificate from a trusted third party.

Request credentials examples include:
* An authentication bearer token,
* A secret session identifier.

Up to now, the basic way to store request credentials in a Web App, was through
the use of sessions: The server requires the client to send an opaque session ID
with each and every request. The server was storing information such as "Who the
user for this session is?", "What is their current CSRF token?", "What is their
current language?". and so on.

However, this is intrinsically [stateful][stateless] (criteria 1). Furthermore,
this solution presents scalability challenges. All the servers providing the
service need to have access to the same session repository. This usually implies
storing the sessions in a database or other complex and error-prone techniques.

In the context of [JSON-LD][json-ld], [JSON Web Tokens][jwt] (JWT) are the _de
facto_ standard and a very elegant solution to the problem. However, a
simple [JSON Web Signature][jws] (JWS) token (as is the current practice) would
allow any attacking party to identify the user by using the stored user ID or
other identifying mean (criteria 10).

A [JSON Web Encryption][jwe] (JWE) token (including both a tamper-proof publicly
readable part and an encrypted part) allows the server to store sensitive
information in the token.

Traditional [JWTs][jwt] only store the user ID or some similar reference to a
user repository hosted on the server. This requires the server to perform a
lookup in this repository for each request. A truly self-contained token usually
requires additional information to be stored in it. For example, the
access-level of the user (or its Roles), User Groups it belongs to, and so on.

In the context of a Web App, these are data that could be useful to the
frontend [SPA][spa] as well. However, in an [HATEOAS][hateoas] context (criteria
1), using API discoverability would provide a leaner way to present these
restrictions. Having API discoverability as the medium for listing which
operations are allowed or not will also prevent caching out of date
authorization data.

### How to design your REST Credentials Container or Token (JWT)

A [JWT][jwt] with the [JWE][jwe] format MUST be used. The data included in the
public part MUST include the following claims:
* `exp`/Expiration date: the client needs to know when to renew their
  credentials.

The data included in the encrypted part MUST include the following claims:
* `aud`/Audience: Origin scope of the token, used for [CSRF][csrf] protection
  (criteria 6);
* `sub`/Subject: the user ID;
* `iss`/Issuer: identification of the trusted third party who provided the user
  authentication (criteria 5);
* `jti`/JWT ID: a unique token ID, used as a nonce to prevent easy
  identification and tracking of the `sub` claim (criteria 10);
* Any additional claims identifying the user or its authorization and
  authentication level.

Any claim whose value is empty or `NULL` MAY be skipped from the token. The
server MUST process such missing claims consistently when verifying the token
validity.

Storing additional data such as the token authentication level, the user's
Roles, or its Groups in the encrypted part of the token aims to prevent
correlating these information in order to track the user (criteria 10).

Short term tokens SHOULD have a lifetime of a few hours at most:
* SHOULD be more than half-an-hour;
* RECOMMENDED value: 1 hour;
* MUST be less than 4 hours.

Long term tokens SHOULD have a lifetime of a few days or weeks:
* SHOULD be more than a week;
* RECOMMENDED value: 1 month;
* MUST be less than a year.

Tokens MUST be renewed explicitly by issuing an API Call whose main purpose is
to renew the token. Resources should not be side-updated and tokens are a
resource like any other.

If a User stays inactive on the frontend App for a time longer than the token
lifetime, the token SHOULD NOT be renewed and the user SHOULD be asked to
re-authenticate themselves.

Short term tokens SHOULD be renewed automatically by the App when a user
interaction occurs after the token has reached half its lifetime. Short term
token SHOULD be renewed only when the user interaction requires a communication
with the server, unless the token is on the brink of expiration (a few minutes
away).

A short term token MUST NOT be renewed or transformed into a long term token
without an explicit request from the user to do so.

A token MAY be demoted (i.e. its associated level of authentication reduced)
when renewed. In particular, an explicitly authenticated short term token SHOULD
be renewed into a remembered short term token. This will prevent users to stay
explicitly authenticated for a long time. If an attacker was able to steal
(e.g. through [XSS][xss]) an explicitly authenticated token, they would have a
limited time to exploit this token to its full potential (criteria 11).

Any request for a long term token (either fresh or renewal) SHOULD carry user
authentication data. This prevents both [CSRF][csrf] and [XSS][xss] attacks
leveraging short term tokens to produce long term ones. This will force the
attacker to refresh a stolen short term token on a regular basis. This technique
increases the resources an attacker needs to spend in order to impersonate a
broad number of users.

A long term/remembered token produced by a request carrying user authentication
MAY be used to represent an explicit authentication for a duration equivalent to
the lifetime of a short term token requested in the same way (minus the
long-term requirement) and at the same time.

A long term token SHOULD be used to generate a remembered short term token at
the beginning of a session. This will reduce the exposition of any long term
tokens and will, therefore, reduce their probability of being stolen.

Long term tokens MAY be renewed automatically once they reached half their
lifetime, like short term tokens. The application SHOULD get confirmation from
the user to renew them.

In the next post, we will focus on [how to store and provide these tokens to the
server][part-2].

[intro]:  /2017/04/22/REST-APIs-authentication-and-security.html                    "Authentication and security for REST API in the context of Web Apps"
[part-2]: /2017/04/23/Credentials-storage-and-transit-in-REST-API-and-web-apps.html "Credentials storage and transit for REST APIs in the context of Web Apps"

[stateless]: https://en.wikipedia.org/wiki/Stateless_protocol "Stateless protocol"
[hateoas]:   https://en.wikipedia.org/wiki/HATEOAS            "Hypermedia As The Engine Of Application State"

[spa]: https://en.wikipedia.org/wiki/Single-page_application "Single-page application"

[json-ld]: http://json-ld.org/                 "JSON for Linking Data"
[jwt]:     https://jwt.io/                     "JSON Web Tokens"
[jws]:     https://tools.ietf.org/html/rfc7515 "JSON Web Signature"
[jwe]:     https://tools.ietf.org/html/rfc7516 "JSON Web Encryption"

[csrf]: https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF) "Cross-Site Request Forgery (CSRF)"
[xss]:  https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)        "Cross-site Scripting (XSS) "
