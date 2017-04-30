---
layout: post
title: "RESTful authentication API for Web Apps (Part 3)"
---

This is the third part of
my [Authentication and security for REST API in the context of Web Apps][intro]
series of posts. In the previous post, we
discussed [how to store and provide tokens to the server][part-2]. In this post,
we will try to design an API to authenticate users and manage tokens.

As mentioned in the introduction, the focus will be on
a [JSON-LD][json-ld], [Hydra][hydra] and [JWT][jwt]-based
implementation. However, this should be easily transposable to any REST
implementation using an alternative for some or all of them.

Unfortunately, not all the [Linked-Data][linked-data] vocabulary we will need is
already available. We will, sometimes, have to define our own vocabulary
terms. In these cases, we will use <http://ld.lemoinem.name/ns/rest-auth#> as
the base IRI for these new terms and `rest-auth:` as their corresponding prefix.
These IRIs are not, currently, populated with linked-data description. In
particular, they are not HTTP dereferenceable, as of today.  However, we intend
to provide such documents once the vocabulary has stabilized and is clear
enough.

This suggested API description is also intended to be presented as
a [JSON-LD][json-ld]/[hydra:ApiDocumentation][hydra:ApiDocumentation] document
describing the designed API at a later date.

Since the [HATEOAS][hateoas] principle's goal is discoverability, it is a given
that the login endpoint/operation IRI MUST be included in the public API
descriptions (such as a [`hydra:ApiDocumentation`][hydra:ApiDocumentation]
referenced by a [`hydra:apiDocumentation`][hydra:apiDocumentationLink] link).

The endpoint SHOULD be documented as a [`hydra:Resource`][hydra:Resource], with
an `@type` appropriate for
a [JWE][jwe] [JWT][jwt]. This [`hydra:Resource`][hydra:Resource], or whatever is
used to represent the authentication endpoint in the API documentation, SHOULD
be given a [`wrds:describedby`][wrds:describedby] property with
value [`rest-auth:authentication`][rest-auth:authentication].

This endpoint's supported operations MUST include:
* one or several `POST` operations on the resource itself, used to create a new
  explicitly authenticated token,
* a `GET` operation, on the resource itself, used to refresh a token.

Since tokens are intended to be opaque and stateless with respect to the server,
`PUT`, `PATCH` and `DELETE` operations MUST NOT be supported. Token are
credentials and, therefore, intrinsically sensitive resources. They require
specific cache handling, which MUST be enforced by the server and respected by
any cache proxy:

* Responses SHOULD have a `Cache-control` Header with the `private` directive,
* Responses SHOULD have a `Cache-control` Header with either:
    * the `max-age` directive with a value less than the the token remaining lifetime,
    * the `must-revalidate` directive
* Responses MUST NOT have a `Cache-control` Header with the `s-maxage`
  directive.
* Responses MUST have a `Vary` Header including "Authentication" (or any other
  HTTP Header used to provide a token to the server) and "Cookie".

This restrictions are intended as a strict minimum. If an implementation chooses
not to follow these recommendations, it MUST provide a stricter cache policy.

The restriction on the `Vary` header is mostly important for unauthenticated or
anonymous resources. This one aside, the same restrictions SHOULD be applied on
any response including an access-restricted authenticated-clients-only resource.

To provide easier automation, responses to the token endpoint SHOULD present
a [`wrds:describedby`][wrds:describedby] link with a
value [`rest-auth:authentication`][rest-auth:authentication].

An unauthenticated `GET` operation (i.e. presented without a token or with an
expired token) on the token resource endpoint MUST return an anonymously
authenticated token.

An anonymously authenticated token is a token with a NULL `sub` claim. All other
relevant claims MUST be filled as with any other token. In particular, `exp`,
`aud` and `jti` are always relevant claims and MUST be filled for all
tokens. For reference, a `NULL` claim MAY be removed from the token.

An authenticated `GET` operation on the token resource endpoint, whose
authentication token has reached half its lifetime MUST send a renewed
token. The renewed token MUST have identical `sub`, `aud` and `iss` claims. The
renewed token MUST have a level of authentication identical to or lower than the
original one (for reference, an explicitly authenticated token SHOULD be renewed
as a remember me token). This renewed token MUST have
a [`rest-auth:use-cookie`][rest-auth:use-cookie] claim equivalent to the old
one.  This renewed token MUST have a different `jti` claim than the old one.
This renewed token's `jti` claim SHOULD be different from any other `jti` claim
included in any previously issued token (and in particular, for the same
user). A cached token revalidation by the server MUST be consistent with this
process.

A `POST` operation on the token resource endpoint MUST provide enough
information to start a credentials validation process. Once the process is
completed successfully, a fresh explicitly authenticated token MUST be returned
in the appropriate format (and transit mode). The response including this new
token SHOULD include a `Content-Location` header containing the IRI of the token
endpoint itself. This MUST be considered a state modifying operation in the
context of [CSRF][csrf] protection to prevent [Login CSRF][login-csrf] attacks.

Once a user agent or a Web App has been through the authentication process and
has been provided a new token, it SHOULD revalidate
the [hydra:ApiDocumentation][hydra:ApiDocumentation] resource as provided in
the [hydra:apiDocumentation][hydra:apiDocumentationLink] link. This is to ensure API
discoverability in the case where the Web App has been hiding its authenticated
API.

The `POST` operations on the token resources MUST support the following inputs
to achieve the following features. Each input SHOULD be tagged (in the API
documentations) with the appropriate value for
its [`wrds:describedby`][wrds:describedby] property. These inputs MAY be
specified as query-string parameters.

* [`rest-auth:use-cookie`][rest-auth:use-cookie]: This input's goal is to
  generate a cookie stored token. This MUST NOT be used by an agent who support
  more RESTful appropriate token storage technology. The token generated by this
  authentication process MUST include an
  encrypted [`rest-auth:use-cookie`][rest-auth:use-cookie] claim set to `true`.
  It MUST be provided to the user agent through a cookie including the
  `httpOnly` flag. This cookie SHOULD include the `secure` flag as well. If the
  API supports non-HTTPS requests, which it SHOULD NOT, the `secure` flag MAY be
  omitted.
* [`rest-auth:remember-me`][rest-auth:remember-me]: This input's goal is to
  generate a long-term Remember Me token. The token generated by this
  authentication process MUST NOT have an higher authenticated level than the
  "Remember Me" level. The user agent SHOULD request a short-term token (which
  SHOULD be marked as explicitly authenticated) as soon as possible after
  receiving a long-term token.

Since the token based authentication is stateless for the server, no logout
endpoint is required. Logging out of the API simply means dropping the token and
starting anew.

### Unauthenticated state modifying operations

In order to prevent [CSRF][csrf] vulnerabilities, the API SHOULD reject with a
"401 Unauthorized" response any unauthenticated (i.e. not including any token)
state modifying request. The `WWW-Authenticate` header MUST include a `Bearer`
challenge whose realm parameter MAY be set to the IRI of the token
endpoint.

Most services nowadays do not offer state modifying request to anonymous users
(an exception could be a completely open and free platform such as Wikipedia).

In particular, the API MUST NOT generate an automatically managed out-of-band
(e.g., in a cookie) non-anonymous authentication token in response to an
unauthenticated (i.e. not including any token) request.

It is very important to consider that some authentication schemes (e.g., the
ones relying on a trusted third party) could come with their own vulnerabilities
to [login CSRF][login-csrf] even for manually managed authentication tokens.

The risk of [login CSRF][login-csrf] is greatly reduced for Web App using a
manually managed token. However, the API must protect Web App using cookies (or
another automatically managed out-of-band method) by ensuring they won't issue
an unrequested authentication token.

The `WWW-Authenticate` header is not the preferred method of discoverability
because it is very loosely defined. User agents SHOULD rely on [HATEOAS][hateoas]
discoverability (e.g. using [Hydra][hydra]) rather than the `WWW-Authenticate`
header until it is better integrated to the [HATEOAS][hateoas] principle.

### Validating a token

In the context of our API, a token is valid if:

1. Its integrity can be verified (both for the public and private parts),
2. It is not expired,
3. It is not used before the moment in time specified by its `nbf` claim, if
   any,
4. It is provided through the correct transit method:
    1. A token with a [`rest-auth:use-cookie`][rest-auth:use-cookie] claim set
       to `true` MUST have been provided as an `httpOnly` cookie. The cookie
       MUST have the `secure` flag unless the API allows non-HTTPS requests.
    2. A token without a [`rest-auth:use-cookie`][rest-auth:use-cookie] claim or
       with such a claim set to `false` or `NULL` MUST have been provided as a
       `Bearer` token through the `Authentication` header.

Additional properties of the token MAY be verified, such as [CSRF][csrf]
protection check or validating the credentials with the token's issuer (when
there is a non-`NULL` `iss` claim).

### [CSRF protection][csrf]

In order to mitigate [CSRF][csrf] vulnerabilities, several checks SHOULD be done
before accepting a token.

The protections against [CSRF][csrf] detailed here are based on the following
assumptions:
* A user-agent will not reuse authentication tokens across different origins,
* A user-agent will not silently switch its credential store to or from cookies
  during a single session,
* A user-agent will consistently publicize the origin of its requests to the server.

"consistently publicize the origin" does not mean a user-agent must send a
`Referrer` or `Origin` Header to the server. It has complete and total freedom on
this point. However, it MUST apply this choice consistently during a single
session.

In particular, the API MUST include the user-agent's `Origin` as the value for
the `aud` claim of any fresh token unless the user-agent's `Origin` is
`NULL`. Renewed tokens MUST carry forward the `aud` claim of their older token.

These [CSRF][csrf] checks only apply if the user-agent provides a
token. Unauthenticated request cannot be protected against [CSRF][csrf] attacks.

[CSRF][csrf] protection is provided by matching the the user-agent's `Origin`
with the the token's `Origin`.

[CSRF][csrf] protection MUST be double-checked for every state modifying
request. [CSRF][csrf] protection SHOULD be double-checked for every
authenticated request.

In order to protect browser user-agent, it is expected that user-agents not
requiring [CSRF][csrf] protection (such as a command-line API client), will not
provide any `Origin` or will craft and consistently provide an appropriate
`Origin` that couldn't be generated by a browser user-agent.

This is not a protection against token theft since, an attacker able of stealing
a token will usually be able to determine the token's original user-agent's
`Origin` easily.

#### Determine the user-agent's `Origin`

First, the server MUST determine the user-agent's `Origin` using this algorithm:

1. If the user-agent provides an `Origin` Header in its request, the value of
   this Header is the user-agent's `Origin`.
2. If the user-agent provides a `Referee` Header in its request and the value of
   this Header is an IRI, the `Origin`'s components of its value (usually the
   scheme, host and port parts of the IRI).
3. Otherwise, the user-agent's `Origin` is `NULL`.

#### Determining the token's `Origin`

The token's `Origin` is the value of its `aud` claim.

### Hiding authenticated API documentation

For security and privacy reasons, an API MAY wish to hide its discoverable API
to unauthenticated user agents.

In this case, any unauthenticated or anonymously authenticated request to the
API MUST provide a [`hydra:apiDocumentation`][hydra:apiDocumentationLink] link to a
curated [`hydra:ApiDocumentation`][hydra:ApiDocumentation]. This unauthenticated
API Documentation MAY use the same IRI as the resource for the authenticated
one. In this case, the server MUST include a `Vary` Header with "Authentication"
(or any other HTTP Header used to provide a token to the server) and "Cookie" in
the response.

This is to ensure caches will not be confused and the authenticated API remains
discoverable easily.

If the authenticated and
unauthenticated [`hydra:ApiDocumentation`][hydra:ApiDocumentation]'s IRIs are
different, the authenticated API Documentation MUST include at least the same
information as the unauthenticated one. Moreover, any response to an
authenticated request MUST reference the authenticated ApiDocumentation.

Several different [`hydra:ApiDocumentation`][hydra:ApiDocumentation] resources
MAY be provided this way, each for a different level of authentication. However,
this might end up confusing both users and user agents. Therefore, API SHOULD
refrain from doing so.

In the next part, we will focus
on [adapting several user authentication schemes to this API][part-4].

[intro]:   /2017/04/22/REST-APIs-authentication-and-security.html                    "Authentication and security for REST API in the context of Web Apps"
[part-2]:  /2017/04/23/Credentials-storage-and-transit-in-REST-API-and-web-apps.html "Credentials storage and transit for REST APIs in the context of Web Apps"
[part-4]:  /2017/04/27/Modular-REST-authentication-API-for-Web-App.html              "Modularity of RESTful authentication API"

[hateoas]:      https://en.wikipedia.org/wiki/HATEOAS  "Hypermedia As The Engine Of Application State"
[linked-data]:  http://linkeddata.org/                 "Connect Distributed Data across the Web"
[json-ld]:      http://json-ld.org/                    "JSON for Linking Data"
[jwt]:          https://jwt.io/                        "JSON Web Tokens"
[jwe]:          https://tools.ietf.org/html/rfc7516    "JSON Web Encryption"
[hydra]:        http://www.markus-lanthaler.com/hydra/ "Hypermedia-Driven Web APIs"

[csrf]:                https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)                                          "Cross-Site Request Forgery (CSRF)"

[login-csrf]: http://www.adambarth.com/papers/2008/barth-jackson-mitchell-b.pdf "PDF: Robust Defenses for Cross-Site Request Forgery"

[hydra:ApiDocumentation]:     http://www.w3.org/ns/hydra/core#ApiDocumentation "The Hydra API documentation class"
[hydra:apiDocumentationLink]: http://www.w3.org/ns/hydra/core#apiDocumentation "A link to the API documentation"
[hydra:Resource]:             http://www.w3.org/ns/hydra/core#Resource         "The class of dereferenceable resources"

[wrds:describedby]: http://www.iana.org/assignments/relation/describedby "Refers to a resource providing information about the link's context."

[rest-auth:authentication]: http://ld.lemoinem.name/ns/rest-auth#authentication "Describes a RESTful authentication API's endpoint"
[rest-auth:use-cookie]:     http://ld.lemoinem.name/ns/rest-auth#use-cookie     "Identifies a cookie stored token"
[rest-auth:remember-me]:    http://ld.lemoinem.name/ns/rest-auth#remember-me    "Identifies a Remember Me token"
