---
layout: post
title: "Modular RESTful authentication API for Web Apps (Part 4)"
---

This is the forth part of
my [Authentication and security for REST API in the context of Web Apps][intro]
series of posts. In the previous post, we designed
a [RESTful authentication API][part-3]. In this post, we will integrate several
user authentication schemes to it (criteria 5).

As for the previous post, <http://ld.lemoinem.name/ns/rest-auth#> is used as the
base IRI for new [Linked-Data][linked-data] vocabulary terms and `rest-auth` is
used as prefix for such terms.

It is important to note that some authentication schemes that have been
traditionally implemented as single step processes (such as user/password
authentication) have been switched to two or multi-steps processes in recent
years for security reasons.

While a multi-steps process is not, a priori, RESTful, we offer an API behavior
that would in fact remove any hidden state.

However, having API operations that are designed to be called in a
pre-determined sequence could confuse user-agents and users, and generate
increased error rates. To solve this problem, implementations SHOULD annotate
their API Documentations with explicit relationships between the calls. For
example, the [Service Ontology][service-ontology] could be used to achieve that.

### Password-based authentication

Currently, the simplest and most common authentication mechanism is password
based authentication: The user provides the server with a username and a
password, if they match what the server has in its database, the credentials are
accepted and the user is authenticated.

The way it is implemented on most websites however is rather lacking. Many
websites do not perform [CSRF][csrf] protection on the login form, and the vast
majority do not offer the user a chance to identify the platform before filling
in their password, thus making the user vulnerable to phishing attempts
(criteria 8).

[Login CSRF][login-csrf] allows an attacker to login the victim in the
attacker's own account. Any information the victim would then fill in, will be
saved in the attacker's account. If the attacked platform is linking accounts
with sensitive platforms or data (e.g., credit card information) this might be a
very serious flaw.

Since our API already request authentication operations to be protected
from [CSRF][csrf] this situation will be averted.

Platform identification is usually the responsibility of an Extended-Validation
TLS certificate. However, such certificates are expensive, can be hacked, and
most users do not check them. Other platforms require the user to choose a
"secret" image and a "personal" phrase (thereafter, anti-phishing proof). These
should be changed on a regular basis and are shown back to the user after they
entered their login but before they enter their password.

This is an excellent solution and extremely easy to implement. In particular in
an [SPA][spa] this should be easy to do without overwhelming the user with
multiple login steps. For example, the anti-phishing proof could be fetched and
shown automatically to the user when they remove the focus from the login field.

To support this authentication scheme, the API MUST provide two `POST`
operations:
1. The first one MUST require only the user's public credentials (login name) as
   input (which is usually a nick name or email address).
2. The second one MUST require a [JWS][jws] [JWT][jwt] and the user's secret
   credentials (password) as input.

The API MUST respond to the first operation with a very short term,
signed [JWS][jws] [JWT][jwt]. This token MUST NOT be used to replace the current
credential token. This token MUST NOT be stored in any permanent storage. This
token `exp` claim SHOULD grant the token a lifetime of a few minutes at
most. This token `sub` claim MUST have the value the user provided as a
login. This token MUST include appropriate private claims to present the user
with their anti-phishing proof. This token MUST include private claims copying
the value of the [`rest-auth:use-cookie`][rest-auth:use-cookie]
and [`rest-auth:remember-me`][rest-auth:remember-me] inputs.

This token is not intended to be used as API credentials, but only to provide
the equivalent of a [nonce][nonce], thus making [replay attacks][replay-attack]
impossible.

Additionally, this token SHOULD be signed, and not simply granted a HMAC code.
This additional restriction allows an automated agent to fetch the API's public
key and double check the signature. This will ensure the server is truly the one
who provided the token. This automated check MUST NOT replace the user's own
verification of the anti-phishing proof without further check on the server
signature (e.g., ensuring the certificate used to sign the token is actually
well known to be associated with the intended service).

Other data than secret images and personal phrase MAY be used as anti-phishing
proof, as long as they enable the user to double check (manually or
automatically) that they are actually in communication with the service they
initially opened an account with.

The client is then expected to invoke the second operation with both the token
provided to them as a result of the first operation and the user's secret
credentials. The API MUST then process the user's credentials and accept or
reject these credentials.

If the API accepts the credentials, it should return an
appropriate [JWE][jwe] [JWT][jwt] credentials token.

The [`rest-auth:use-cookie`][rest-auth:use-cookie]
and [`rest-auth:remember-me`][rest-auth:remember-me] inputs provided to the
second operation MUST match the ones provided to the first operation. The API
SHOULD ignore the values provided with the second operation if they don't match.

These passwords SHOULD be required to be appropriately strong.

### [One-time passwords][otp]

These time or counter dependent passwords are getting more and more common. They
are typically used for Multi-Factor Authentication.

In this case, we actually propose to offer them as a primary secret credential
for authentication. Once users are getting more used to them, they are usually
harder to loose than a password. Most users will more easily forget their
password than, say, loose their smartphones. Moreover, we remove the need to
create and remember a password, which is often the most problematic (both UX-
and security-wise) step in the account creation process.

The API requirements are exactly the same as for password-based authentication,
except instead of expecting a password as the second operation's input, the API
should expect the one-time password.

The API MAY provide the two authentications schemes:
* through a unified set of operations.
* through a unified first operation, but using two separate second-step
  operations.
* through completely separated sets of operations.

If the API chooses to use a unified set of operations, it MAY use pattern-based
restrictions to differentiate one time password from traditional password. For
example, most one time passwords are a sequence of six-to-eight digits. Any
traditional password that is such a sequence SHOULD therefore be forbidden.

### Other two-parties single or multi-factor authentication schemes

Many other authentication schemes can be used with similar same operation sets.
Adding secret credential inputs to the second operation allows easy Multi-Factor
Authentication.

Many other authentication schemes requiring a single or two round-trips to the
server (e.g., [SRP][srp]) could be implemented (more or less seamlessly) in
addition or replacement of the traditional or one time password schemes.

### External trustee authentication schemes

External trustee authentication schemes such
as [OpenID][openid], [OAuth2][oauth2], or [SAML][saml] can be easily integrated
with this API.

The simplest way would be to provide a `POST` operation for each such scheme. The
OAuth provider or OpenID's URL/XRI could be provided as the operation's input.

As these authentication schemes rely on an external trustee, this means the Web
App will be unloaded and reloaded with the trustee's information.

It is very important to protect the part of the app loading the trustee's
information against [Login CSRF][login-csrf].

To this end, the initial `POST` operation, providing the API with the user
selection of the trustee MUST provide a return URL containing the
anonymous [JWT][jwt]'s `jti` as a [CSRF][csrf] token. When returning from the
trustee's platform, the Web App MUST send:
* The trustee's authentication data,
* The anonymous [JWT][jwt],
* The [CSRF][csrf] token sent as part of the trustee's callback.

If the `jti` of the anonymous [JWT][jwt] and the [CSRF][csrf] token do not
match, the API MUST NOT issue an authenticated token. It SHOULD instead return a
"400 Bad Request" Response. This along with the usual [CSRF][csrf] protection
already granted by our [JWT][jwt] token ensures the API and Web App will not
accept unrequested credentials tokens.

Our API is now able to integrate with widely different, including the most
common, user authentication scheme. However, there are
some
[security flaws that need to be further discussed in order to be more thoroughly mitigated against][part-5].

[intro]:  /2017/04/22/REST-APIs-authentication-and-security.html "Authentication and security for REST API in the context of Web Apps"
[part-3]: /2017/04/23/REST-authentication-API-for-Web-App.html   "RESTful authentication API"
[part-5]: /2017/04/29/Securing-a-RESTful-authentication-API.html "Security of a RESTful authentication API"

[linked-data]: http://linkeddata.org/              "Connect Distributed Data across the Web"
[jwt]:         https://jwt.io/                     "JSON Web Tokens"
[jws]:         https://tools.ietf.org/html/rfc7515 "JSON Web Signature"
[jwe]:         https://tools.ietf.org/html/rfc7516 "JSON Web Encryption"

[spa]: https://en.wikipedia.org/wiki/Single-page_application "Single-page application"

[otp]:    https://en.wikipedia.org/wiki/One-time_password "One-time password"
[srp]:    http://srp.stanford.edu/                        "Secure Remote Password"
[openid]: http://openid.net/                              "OpenID"
[oauth2]: https://oauth.net/2/                            "OAuth 2.0"
[saml]:   https://wiki.oasis-open.org/security/FrontPage  "SAML V2.0"

[csrf]:          https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)    "Cross-Site Request Forgery (CSRF)"
[login-csrf]:    http://www.adambarth.com/papers/2008/barth-jackson-mitchell-b.pdf    "PDF: Robust Defenses for Cross-Site Request Forgery"
[nonce]:         https://www.owasp.org/index.php/Glossary#Nonce                       "Nonce"
[replay-attack]: https://www.owasp.org/index.php/Testing_for_WS_Replay_(OWASP-WS-007) "WebService Replay attack"

[service-ontology]: https://dini-ag-kim.github.io/service-ontology/service.html "The Service Ontology"

[rest-auth:use-cookie]:  http://ld.lemoinem.name/ns/rest-auth#use-cookie  "Identifies a cookie stored token"
[rest-auth:remember-me]: http://ld.lemoinem.name/ns/rest-auth#remember-me "Identifies a Remember Me token"
