---
layout: post
title: "Securing a RESTful authentication API for Web Apps (Part 5)"
---

This is the fifth part of
my [Authentication and security for REST API in the context of Web Apps][intro]
series of posts. In the previous post,
we
[integrated various user authentication scheme to our authentication API][part-4]. In
this post, we will focus on mitigating several important security issues.

Although our authentication API includes protections
against [login CSRF][login-csrf] attacks, there are other attacks that could be
triggered easily. Let's try to expose such attacks and find a way to mitigate
them.

### [Phishing attacks][phishing] (criteria 8)

Using the anti-phishing proof technique prevents an attacker to easily
impersonate the API and grab a user's credentials. However, it doesn't make such
an attack impossible, but only requires the attacker to communicate with the
actual API in order to present the user with correct information.

For these reasons, access to the API should be appropriately restricted:
* An API which is not intended to be accessible by non-browser clients SHOULD
  require a user-agent to behave similarly to a standard browser and to provide
  an `Origin`.
* An API which is intended to be accessed only through a small set of in-browser
  clients SHOULD use Origin-restrictions such as [CORS policies][cors] to
  white list authorized `Origin`s.

Our requirement for consistent user-agent's `Origin`s already segregate
browser-like user-agents from programmatic or native software-like
user-agents. Special care should be taken as to which browsers and environments
are supported. Some networks or user-agents might be configured to block
`Origin` and `Referred` Headers, this must be taken into account while applying
restrictions on the API.

Even then, these restrictions will never prevent completely a phishing
attack. Our goal is to require the attacker to provide its own system proxying
requests to the real API and impersonating a browser-like user agent acting from
the correct `Origin`. Additional phishing mitigation measures must be put in
place, such as Extended Validation TLS certificate or another kind of public
verification of identity provided by a trusted third-party.

Phishing attacks can never be fully prevented at the application level. Like
every social engineering based attacks, the most effective prevention measures
are user education and public dissemination of identity information.

### [Account enumeration][account-enumeration] (criteria 9)

An account enumeration attack is an attack trying to leverage answers from the
authentication process to determine whether an account exists or not. These
attacks allow an attacker to reduce the attack surface and focus on account that
actually exist.

They are the reason why most websites nowadays won't tell the user whether they
made a typo in their username or in their password. The user is usually
presented with a generic "Invalid login or password". This forces an attacker to
confirm the existence of an account using other indirect and more costly means.

Since our API is designed to authenticate the server by providing the user with
an anti-phishing proof, it would be trivial to know whether an account exists:
If the API is not able to provide a anti-phishing proof, then the account does
not exist.

A way to mitigate these attacks would be to return uniform answers to the first
step of the authentication process.

Whether an account exists or not, the API MUST return a "200 OK" answer, with an
anti-phishing proof. If the account does not exist, the anti-phishing proof MUST
be generated or picked randomly. It MUST be based on a representative sample of
actual anti-phishing proofs.

An easy way to implement this technique is not to offer an unrestricted choice
to the user for its anti-phishing proof. By offering a pick in a predetermined
bank (such as a bank of stock photos and a bank of quotes from well known works
of literature), the API can easily generate fake anti-phishing proofs.

Theses choice banks SHOULD be bigger (twice or thrice) than the actual amount of
expected users in the system. They SHOULD be offered to the user as a random
sample in a random order. This will ensure collisions (accounts with the same
anti-phishing proof) will be limited and each phrase or image has about half or
a third probability to be used by a real account, leaving plenty of choices to
generate fake anti-phishing proof from.

This will actually mitigate only direct probing. An attacker could still query
twice in a row each account. If an account gives two different anti-phishing
proofs, then the account does not exist.

For this reason, the last generated anti-phishing proofs and their associated
login name SHOULD be cached by the server. For example, a platform with 5,000
accounts could store up to 2,500 generated "fake" accounts, each for up to 30
minutes.  This will ensure no attacker can check the existence of an account by
probing it twice (or any number of times) in a row. Limiting the lifetime and
number of cached fake accounts prevents a single attacker to force the server to
store a dangerous amount of useless data.

An attacker can still probe indirectly the accounts, either by checking accounts
in batch of thousands or by waiting 30 minutes between probes on a same
account. In any case, both the number of cached fake account and the cache
duration can be adapted to force an appropriate cost on the attacker.

For platforms with a very large or quick growing number of users, having a
pre-determined bank of images and phrases might be impractical. In these case,
the platform could leverage machine learning techniques. This means training
regularly and automatically an anti-phishing proof generator on the (anonymized)
existing ones. This generator could then be used to generate anti-phishing
proofs that would be indistinguishable from actual ones. This follows the
principle that more complex platforms will require more complex security
processes.

As for every technique conflating two types of answers or errors, additional care
must be taken to avoid [timing attacks][timing-attack].

### [User tracking][privacy] (criteria 10)

Our authentication token is designed in such a way that it will never provide
information that could allow tracking the user. However, APIs typically offer an
endpoint allowing a user to fetch and update their own account.

An attacker able to issue requests impersonating the user (e.g.,
through [XSS][xss], [CSRF][csrf] or [token theft][session-hijacking] attacks)
could query this endpoint and use the answer to track the user. For these
reasons, access to such an endpoint SHOULD be restricted to explicitly
authenticated tokens.

Often, a user will require access to information on other accounts. Either to
see who they are interacting with through the platform, or to manage accounts
for which they have a supervisor role.

Restricting read access to other accounts to explicitly authenticated tokens
might quickly force the user to always be explicitly authenticated. This would
make remember me tokens useless and is, therefore, NOT RECOMMENDED. However,
write access to any account SHOULD be restricted to explicitly authenticated
tokens because of the inherent security implications.

Obviously, different platforms will have different requirement regarding:
* Whether long or short term remember me tokens are available?
* Whether explicitly authenticated tokens should be automatically demoted?
* What is available to each kind of token?

These are important security questions that need to be answered on a
case-by-case basis.

Our authentication API is now ready and secure enough to be used. The next and
last post will focus on
other
[miscellaneous security issues commonly found in Web Apps or APIs][part-6].

[intro]:  /2017/04/22/REST-APIs-authentication-and-security.html       "Authentication and security for REST API in the context of Web Apps"
[part-4]: /2017/04/27/Modular-REST-authentication-API-for-Web-App.html "Modularity of RESTful authentication API"
[part-6]: /2017/04/29/Misc-security-concerns-for-REST-APIs.html        "Other security consideration for RESTful APIs"

[cors]: https://www.w3.org/TR/cors/ "Cross-Origin Resource Sharing"

[csrf]:                https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)                                          "Cross-Site Request Forgery (CSRF)"
[xss]:                 https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)                                                 "Cross-site Scripting (XSS) "
[phishing]:            https://www.owasp.org/index.php/Phishing                                                                   "Phishing"
[account-enumeration]: https://www.owasp.org/index.php/Testing_for_Account_Enumeration_and_Guessable_User_Account_(OTG-IDENT-004) "Account Enumeration"
[session-hijacking]:   https://www.owasp.org/index.php/Session_hijacking_attack                                                   "Session hijacking attack"

[login-csrf]: http://www.adambarth.com/papers/2008/barth-jackson-mitchell-b.pdf "PDF: Robust Defenses for Cross-Site Request Forgery"

[timing-attack]: https://en.wikipedia.org/wiki/Timing_attack "Timing attack"
