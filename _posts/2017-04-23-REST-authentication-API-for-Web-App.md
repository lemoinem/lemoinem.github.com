---
layout: post
title: "RESTful authentication API for Web Apps (Part 3)"
---

This is the third part of my "[Authentatication and security for REST API in the
context of Web Apps](/2017/04/22/REST-APIs-authentication-and-security.html)"
series of posts. Now, that we discussed how to design, store and send your credentials token, we will
try to find an API to manage and authenticate users.

We will start by focusing on how to provide a nice RESTful authentication (token management) API.

As mentioned in the introduction, my focus will be on a [JSON-LD](https://www.w3.org/TR/json-ld/),
[Hydra](http://www.hydra-cg.com/spec/latest/core/) and [JWT](https://tools.ietf.org/html/rfc7519)-based
implementation. However, this should be easily transposable to any alternative to some or all of them.

Unfortunately, not all the vocabulary we will need is already available.
We will, sometimes, need to define our own vocabulary terms. In these cases,
we will use <http://ld.lemoinem.name/ns/rest-auth#> as the base IRI for this new vocabulary.
These IRIs are not, currently, populated with linked-data description. In particular,
they are not HTTP dereferenceable for now.
However, I intend to provide such documents once the vocabulary has stabilized and is clear enough.

This suggested API description is also intended to be presented as a
JSON-LD/[hydra:ApiDocumentation](http://www.w3.org/ns/hydra/core#ApiDocumentation) document at a later date.

Given that RESTful HATEOAS APIs are all about discoverability, it is a given that the login endpoint/operation IRI MUST
be included in the public API descriptions (such as a
[hydra:ApiDocumentation](http://www.w3.org/ns/hydra/core#ApiDocumentation)
referenced by a [hydra:entrypoint](http://www.w3.org/ns/hydra/core#entrypoint) Link).

For now, the best way I found to ensure automatic discoverability of this endpoint,
is to provide it as a [hydra:Class](http://www.w3.org/ns/hydra/core#Class), describing a JWE JWT token.

To provide easier automation, this [hydra:Class](http://www.w3.org/ns/hydra/core#Class) SHOULD be given a
[wrds:describedBy](http://www.iana.org/assignments/relation/describedby)
property with value [rest-auth:authentication](http://ld.lemoinem.name/ns/rest-auth#authentication).

The supported operations MUST include:
* one or several POST operations on the resource itself, used to create a new explicitly authenticated token,
* a GET operation, on the resource itself, used to refresh a token.

Since tokens are intended to be opaque and stateless with respect to the server,
PUT, PATCH and DELETE operations MUST NOT be supported.
Token being credentials based, intrinsicaly sensitive, resources, they require specific cache handling, which MUST be
enforced by the server and respected by any cache proxy:

* Responses SHOULD have a Cache-control Header with the `private` directive,
* Responses SHOULD have a Cache-control Header with either:
    * the `max-age` directive with a value less than the the token remaining lifetime,
    * the `must-revalidate` directive
* Responses MUST NOT have a Cache-control Header with the `s-maxage` directive.
* Responses MUST have a Vary Header with "Authentication"
  (or any other HTTP Header used to provide a token to the server) and "Cookie".

This restrictions are intended as a strict minimum. If an implementor choose not to follow these recommendations,
they MUST provide a stricter cache policy.

Except the restriction on the Vary header, which is mostly important for unauthenticated or anonymous resources,
the same restrictions SHOULD be applied on any response including an access-restricted authenticated-only resource.

To provide easier automation, responses to the token endpoint SHOULD present a Link header, using a rel of
[wrds:describedBy](http://www.iana.org/assignments/relation/describedby)
and a value [rest-auth:authentication](http://ld.lemoinem.name/ns/rest-auth#authentication).

An unauthenticated GET (i.e. presented without a token or with an expired token) operation on the
token resource endpoint MUST return an anonymously authenticated token.

An anonymously authenticated token is a token without or with a NULL `sub` claim.
All other relevant claims MUST be filled as with any other token (in particular, `exp`, `aud` if not NULL and
`jti` are always relevant claims and MUST be filled for all tokens).

An authenticated GET operation on the token resource endpoint whose authentication token has reached half its
lifetime or more MUST send a new token, with identical `sub`, `aud` and `iss` claims. This new token MUST have
the same level of authentication or lower as the original one (e.g., an explicitly authenticated token SHOULD
be renewed as a remember me token). This new token MUST have a
[rest-auth:use-cookie](http://ld.lemoinem.name/ns/rest-auth#use-cookie) claim equivalent to the previous one.
This new token MUST have a different `jti` than the previous one.
This new token's `jti` claim SHOULD be different from any other `jti` claim included in any previously
issued token (in particular, for the same user). A cached token revalidation by the server MUST be consistent
with this process.

A POST operation on the token resource endpoint MUST provide enough information to start a credentials
validation process. Once the process is completed successfuly, a fresh explicitly authenticated token
MUST be returned in the appropriate format (and transit mode). The response including this new token SHOULD include
a Content-Location header containing the IRI of the token endpoint itself. This MUST be considered a state modifying
operation in the context of CSRF protection to prevent Login CSRF attacks.

Once a user agent or a Web App has been through the authentication process and has been provided a new token,
it SHOULD revalidate the [hydra:ApiDocumentation](http://www.w3.org/ns/hydra/core#ApiDocumentation) resource as
provided in the [hydra:entrypoint](http://www.w3.org/ns/hydra/core#entrypoint). This is to ensure API discoverability
in the case where the Web App has been hiding its authenticated API.

The POST operations on the token resources MUST supports inputs to achieve the following features.
Each input SHOULD be tagged (in the API documentations) with the appropriate value for the
[wrds:describedBy](http://www.iana.org/assignments/relation/describedby) property. These inputs MAY be
specified as query-string parameters.

* [rest-auth:use-cookie](http://ld.lemoinem.name/ns/rest-auth#use-cookie): This input's goal
  is to generate a cookie stored token. This MUST be used by an agent who does not support more RESTful
  token storage technology. The token generated by this authentication process MUST include an encrypted
  [rest-auth:use-cookie](http://ld.lemoinem.name/ns/rest-auth#use-cookie) claim set to `true`.
  It MUST be provided to the user agent through a cookie including both httpOnly and secure flags.
  If the API supports non-HTTPS requests, which it SHOULD NOT, the secure flag MAY be omitted.
* [rest-auth:remember-me](http://ld.lemoinem.name/ns/rest-auth#remember-me): This input's goal
  is to generate a long-term Remember Me token. The token generated by this authentication process MUST include an
  encrypted [rest-auth:remember-me](http://ld.lemoinem.name/ns/rest-auth#remember-me) claim set to `true`. The user agent
  SHOULD request a short-term token (which SHOULD be marked as explicitly authenticated) as soon as possible after
  recieveing a long-term token.

Since the token based authentication is stateless for the server, no logout endpoint is required. Loging out of the API
simply means droping the token and starting anew.

### Unauthenticated state modifying operations

In order to prevent CSRF vulnerabilities, the REST API SHOULD reject with a "401 Unauthorized" response
any unauthenticated (i.e. not including any token) state modifying request.
The WWW-Authenticate header MUST include a Bearer challenge whose realm parameter MAY be set to the
IRI of the token endpoint. Most services nowadays do not offer state modifying request
to anonymous users (an exception could be a completely open and free platform such as Wikipedia).

In particular, the REST API MUST reject in this manner any unauthenticated (i.e. not including any token)
request to generate an automatically managed out-of-band (e.g., in a cookie) authentication token.

It is very important to consider that some authentication schemes (e.g., the ones relying on
a trusted third party) could come with their own vulnerabilities to login CSRF even
for manually managed authentication tokens.

The risk of login CSRF is greatly reduced for Web App using either an in-band or
manually managed out-of-band authentication token. However, the API must protect Web App using
cookies (or another automatically managed out-of-band method) by ensuring they won't issue
an unrequested authentication token.

The WWW-Authenticate header is not the prefered method of discoverability because it is very lously defined. 
User agents SHOULD rely on HATEOAS discoverability (e.g. using hydra) rather than the WWW-Authenticate header
until it is better integrated to the HATEOAS principle.

### Validating a token

In the context of our API, a token is valid if:

1. Its integrity can be verifies (both for the public and private parts),
2. It is not expired,
3. It is not used before the moment in time specified by its `nbf` claim,
4. It is provided through the correct transit method:
    1. A token with a [rest-auth:use-cookie](http://ld.lemoinem.name/ns/rest-auth#use-cookie) claim set to `true`
       MUST have been provided as an httpOnly cookie. The cookie MUST be secure unless the API allows non-HTTPS requests.
    2. A token without a [rest-auth:use-cookie](http://ld.lemoinem.name/ns/rest-auth#use-cookie) claim or with such a claim
       set to `false` MUST have been provided as a Bearer token through the Authentication header.

Additional properties of the token MAY be verified, such as CSRF protection check or validating the credentials with the 
token's issuer (when there is an `iss` claim).

#### CSRF protection

In order to mitigate CSRF vulnerabilities, several checks should be done before accepting a token.

The protections against CRSF detailed here are based on the following assumptions:
* A user agent will not reuse authentication tokens across different origins,
* A user agent will not silently switch its credential store to or from cookies,
* A user agent will consistently publicize the origin of its requests to the server.

"consistently publicize the origin" does not mean a user agent must send a Referer or Origin Header to the server, it has
complete and total freedom to choose to do so or not. However, it MUST apply this choice consistently during the whole
session's lifetime.

In particular, the service MUST include the user-agent's `Origin` as the value for the `aud` claim of any fresh
token unless the user-agent's `Origin` is NULL.

CSRF checks only apply if the user-agent provides a token. Unauthenticated request cannot be protected against CSRF attacks.

CSRF protection is provided by matching the the user-agent's `Origin` with the the token's `Origin`.

CSRF protection MUST be double-checked for every state modifying request.
CSRF protection SHOULD be double-checked for every authenticated request.

It is expected that user-agents not requiring CSRF protection (such as a command-line API client), will not provide any
Origin or will craft and use consistently an appropriate `Origin` header that couldn't be generated by a browser user-agent.

This is not a protection against token theft since, an attacker able of stealing a token will usually be able
to determine the token's original user-agent's Origin easily.

##### Determine the user-agent Origin

First, the server MUST determine the user-agent's `Origin` using this algorithm:

1. If the user-agent provides an `Origin` Header in its request, the value of this Header is the user-agent's `Origin`.
2. If the user-agent provides a `Referer` Header in its request and the value of this Header is an IRI, the `Origin`'s
   components of its value (usually the scheme, host and port parts of the IRI).
3. Otherwise, the user-agent's `Origin` is `NULL`.

##### Determining the token's `Origin`

The token's `Origin` is the value of its `aud` claim or `NULL` if the claim is absent.

### Hiding authenticated API documentation

For security and privacy reasons, an API MAY wish to hide its discoverable API to unauthenticated user agents.

In this case, any unauthenticated or anonymously authenticated request to the API MUST provide a curated
[hydra:entrypoint](http://www.w3.org/ns/hydra/core#entrypoint). This entry point MAY use the same IRI as
the authenticated one. In this case, the server MUST include a Vary Header with "Authentication"
(or any other HTTP Header used to provide a token to the server) and "Cookie" in the response.

This is to ensure caches will not be confused and the authenticated API remains discoverable easily.

If the authenticated and unauthenticated [hydra:entrypoint](http://www.w3.org/ns/hydra/core#entrypoint)'s IRIs are
different, the authenticated entrypoint MUST include at least the same information as the unauthenticated one. Moreover,
any response to an authenticated request MUST reference the authenticated endpoint.

Several different [hydra:entrypoint](http://www.w3.org/ns/hydra/core#entrypoint) MAY be provided, each for a different level
of authentication. However, this might end up confusing both users and user agents. Therefore, API
SHOULD refrain from doing so.
