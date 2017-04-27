---
layout: post
title: "Modular RESTful authentication API for Web Apps (Part 4)"
---

This is the fourth part of my "[Authentatication and security for REST API in the
context of Web Apps](/2017/04/22/REST-APIs-authentication-and-security.html)"
series of posts. The previous post laid the foundations for a general-purpose CSRF-resistent
authentication REST API.

However, there are still several issues to address in order to make sure the API supports
several authentication schemes.

As for the previous post, <http://ld.lemoinem.name/ns/rest-auth#> is used as the base IRI for new
Linked-Data vocabulary terms.

In the introduction, we mentioned our authentication API should support multiple authentication schemes.

It is important to note that some authentication schemes that have been traditionally single step (such as user/password
authentication) have been switched to a two or multi-steps process in recent years for security reasons.

While a multi-steps process is not, a priori, RESTful, we offer an API behavior that would in fact remove any hidden state.

However, having API operations that are designed to be called in a pre-determined sequence could confuse user-agents
and users and provoque increased error rates. To solve this problem, API SHOULD anotate their API Documentations with
explicit relationships between the calls. For example, the
[Service Ontology](https://dini-ag-kim.github.io/service-ontology/service.html) could be used to achieve that.

### Password-based authentication

Currently, the simplest and most common authentication mechanism is password based authentication:
The user provides the server with a username and a password, if they match what the server has in database,
the credentials are accepted.

The way it is implemented on most website however is rather lacking. Many website do not perform CSRF protection
on the login form, and the vast majority do not offer the user a chance to identify the platform before filling in their
password.

Login CSRF allows an attacker to login the victim in the attacker's own account. Any information the victim would then
fill in, would be saved in the attacker's account. If your website is linking accounts with sensitive platforms or
storing credit cards, for example, this might be a very serious flaw.

Since our API already request authentication operations to be protected by CSRF this situation will be averted.

Platform identification is usually the responsibility of an Extended-Validation TLS certificate. However, such certificates
are expensive, can be hacked and most users do not check them. Other platforms require the user
to choose a "secret" image and a "personal" phrase. These should be changed on a regular basis
and are shown back to the user after they entered their login but before they enter their password.

This is an excellent solution and extremely easy to implement. In particular in an SPA this should be easy to do without
overwhelming the user with multiple login steps. For example, the secret image and personnal phrase could be fetched
and shown to the user when they remove the focus from the login field.

To support this authentication scheme, the REST API MUST provide two POST operations:
1. The first one MUST require only the user's public credentials (login name) as input
   (which is usually a nick name or email address).
2. The second one MUST require a JWS JWT and the user's secret credentials (password) as input.

The API MUST respond to the first operation with a "200 OK" HTTP response (or equivalent), containing
a very short term, signed JWS JWT token. This token MUST NOT be used to replace the
current credential token. This token MUST NOT be stored in any permanent storage. This token `exp` claim
SHOULD grant the token a lifetime of a few minutes at most. This token `sub` claim MUST have
the value the user provided as a login. This token MUST include appropriate private claims to present the user with their
secret image and personal phrase. This token MUST include private claims copying the value of the
[rest-auth:use-cookie](http://ld.lemoinem.name/ns/rest-auth#use-cookie) and
[rest-auth:remember-me](http://ld.lemoinem.name/ns/rest-auth#remember-me) inputs.

This token is not intended to be used as API credentials, but only to provide the equivalent of a nonce. In order to make
replay attacks impossible.

Additionaly, this token needs to be signed, and not simply granted a HMAC code, because this way, an automated agent could
fetch the API's public key and double check the signature. This would ensure the server is truly the one who providing the
token. This automated check MUST NOT replace the user's verification of their secret image and personal phrase.

Other datums than secret images and personal phrase MAY be used, as long as they enable the user to double check (manually
or automatically) some information allowing them to not only authenticate the server itself, but ensure that they are
actually in communication with the service they initially opened an account with.

The client is then expected to invoke the second operation with both the token provided to them as a result of the first
operation and the user's password. The API MUST then process the user's credentials and accept or reject the credentials.

If the API accepts the credentials, it should return an appropriate JWE JWT credentials token.

The [rest-auth:use-cookie](http://ld.lemoinem.name/ns/rest-auth#use-cookie) and
[rest-auth:remember-me](http://ld.lemoinem.name/ns/rest-auth#remember-me) inputs provided to the second operation MUST
match the ones provided to the first operation. The API SHOULD ignore the values provided with the second
operation if they don't match.

Traditional passwords SHOULD be required to be appropriately strong.

### Time-based and/or one-time password

These time dependent passwords (e.g., as generated by Google Authenticator) are getting more and more
widespread and are typically as mean for Multi-Factor Authentication.

In this case, we actually propose to offer them as a primary credential for authentication. Once users are getting more
used to them, they are usually harder to loose than a password, since most users will more easily forget
their password than, say, loose their smartphones. Moreover, we remove the need to create and remember a password,
which is often the most problematic (both UX- and security-wise) step in the account creation process.

The API requirements are exactly the same as for password-based authentication, except instead of expecting a password as
the second operation's input, the API should expect the time-based or one-time password.

The API MAY provide the two authentications schemes:
* through a unified set of operations.
* through a unified first operation, but using two separate second-step operations.
* through completely separated set of operations.

If the API choose to use a unified set of operations, it SHOULD use pattern-based restrictions to differentiate time based
password from traditional password. For example, most time-based passwords are a sequence of six digits. Any traditional
password that is a sequence of six digits SHOULD therefore be forbidden.

### Other single or multi-factor authentication schemes

Many other authentication schemes can be used with the same operation sets.
Adding secret credential inputs to the second operation allows easy Multi-Factor Authentication.

Many other authentication schemes requiring a single or two round-trips to the server
(e.g., [SRP](http://srp.stanford.edu/)) could be implemented (more or less seamlessly) in addition or
replacement of the traditional or time-based password schemes.

### External trustee authentication schemes

External trustee authentication schemes such as OpenID, OAuth2, or SAML can be easily integrated with the API.

The simplest way would be to provide a POST operation for each such scheme. The OAuth provider or OpenID's
URL/XRI could be provided as the operation's input.

As these authentication schemes rely on an external trustee, this means the Web App will be unloaded
and reloaded with the trustee's information.

It is very important to protect the part of the app loading the trustee's information against Login CSRF.

To this end, the initial POST operation, providing the API with the user selection of the trustee MUST provide a return URL
containing the anonymous JWT's `jti` as a CSRF token. When returning from the trustee's platform, the Web App MUST send:
* The trustee's authentication data
* The anonymous JWT
* The CSRF token send as part of the trustee's callback

If the `jti` of the anonymous JWT and the CSRF token do not match, the API MUST NOT issue an authenticated token,
but SHOULD return a 400 Bad Request HTTP Response. This along with the usual CSRF protection already granted by our
JWT token ensures the API and Web App will not accept unrequested JWT tokens.
