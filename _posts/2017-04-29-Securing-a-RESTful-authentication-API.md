---
layout: post
title: "Securing a RESTful authentication API for Web Apps (Part 5)"
---

This is the fifth part of my "[Authentatication and security for REST API in the
context of Web Apps](/2017/04/22/REST-APIs-authentication-and-security.html)"
series of posts. Up to now, we have designed together a secure authentication token
and a RESTful authentication API to manage these tokens.

Although our authentication API includes protections against CSRF login attacks, there are other attacks that
could be triggered easily. Let's try to expose such attacks and find a way to mitigate them.

### Phishing attacks

Using the secret image and personal phrase technique prevent an attacker to easily impersonate the API and grab a user's
credentials (phishing attacks). However, it doesn't make such an attack impossible, but only requires the attacker
to communicate with the actual API in order to present the user with correct information.

For these reasons, access to the API should be appropriately restricted:
* An API which is not intended to be accessible by non-browser clients SHOULD require a user-agent to behave similarly to
  a standard browser and to provide an Origin or a Referer header.
* An API which is intended to be accessed only through a small set of in-browser clients SHOULD use Origin-restrictions such   as CORS policies to whitelist authorized Origins.

Our requirement for consistent user-agent origins already segregate browser-like user-agents from programatic or
native software-like user-agents. Special care should be taken as to which browsers and environments are supported.
Some networks or user-agents might be configured to block Origin and Refered Headers, this must be taken into account
while applying restrictions on the API.

Even then, these restrictions will never prevent completely phishing attacks. However, they will require the attacker
to provide its own system proxying requests to the real API and impersonating a browser-like user agent acting from the
correct Origin. Additional phishing-mitigating measures must be put in place, such as Extended Validation TLS certificate
or another kind of publiic verification of identity provided by a trusted third-party.

These attacks can never be fully prevented at the application level. Like every social engineering based attacks, the
most efficient prevention measures are user education and identity information public dissemination.

### Account probing

An account probing is an attack trying to leverage answers from the authentication process to determine whether an account
exists or not. These attacks allow an attacker to reduce the attack surface and focus attack vectors.
They are the reason why most websites nowadays won't tell the user whether an account doesn't exist or a given
password for an existing account is invalid. The user is usually presented with a generic "Invalid login or password".
This forces an attacker to confirm the existence of an account using other indirect and more costly means.

Since our API is designed to authenticate the server by providing the user with a secret image and personal phrase,
it would be trivial to know whether an account exists: If the API is not able to provide a secret image and
personal phrase, then the account does not exist.

A way to mitigate these attacks would be to return uniform answers to the first step of the authentication process.

Whether an account exists or not, the API MUST return a 200 OK answer, with a secret image and personal phrase.
If the account does not exist, they MUST be generated or picked randomly, but based on a representative
sample of actual secret images and personal phrases.

An easy way to implement this is not to offer an unrestricted choice to the user for its image and phrase, but to
offer a pick in a predetermine bank (such as a bank of stock photos and a bank of quotes from well known
works of litterature).

Theses banks should be bigger (twice or thrice) than the actual amount of expected (not real) users in the system
and be offered to the user as a random sample in a random order.
This will ensure collisions will be reduced extensively and each phrase of image has about a one-in-two or one-in-three
chances to be used by a real account.

This will actually mitigate direct probing. The answer for a given login name SHOULD be cached by the server for a
limited amount of time with a limit on the number of fake accounts memoized based on a
proportion of the real number of accounts. For example, a platform with 5,000 accounts could store up to 1,000
generated "fake" accounts, each for up to 30 minutes.
This will ensure no attacker can check the existence of an account by probing
it twice in a row and checking whether the image and phrase changed. But, at the same time, it will prevent a single
attacker forcing the server to store a dangerous amount of useless data. 

An attacker can still probe indirectly the accounts, either by checking accounts in batch of a thousand or by waiting 30
minutes between probes on a same account. In any case, both the number of cached account and the cache duration can be
adapted to force an adapted cost on the attacker.

For platforms with a very large or quick growing number of users, having a pre-determined bank of images and
phrases might be impractical. In these case, the platform could leverage machine learning techniques
to regularly and automaticaly train an image and phrase generator on the (anonymized) images and phrases
chosen by there actual accounts. This generator could then be used to generate phrases and images that
would appear similar or indistinguishable from an actual one. This is in check with the fact that more complexe platforms
will require more complexe security processed.

As for every technique conflating two types of answers or erros, additional care must be taken to avoid timing attacks.

### Account tracking

Our authentication token is designed in such a way that it will never provide information that could allow
tracking the user. However, APIs typically offer an endpoint allowing a user to fetch and update their own account.
An attacker able to issue requests impersonating the user (e.g., through XSS, CSRF or token-theft attacks) could query
this endpoint and use the answer to track the user. For this reason, access to such an endpoint SHOULD be restricted
to explicitly authenticated tokens.

Often, a user will require access to information on other accounts, either to see who they are interacting with through
the platform or to manage accounts for which they have a supervisor role.

Restricting read access to other accounts to explicitly authenticated tokens only might quickly force the user
to always be explicitly authenticated. This would make remember me tokens useless and is NOT RECOMMENDED.
However, write access to any account SHOULD be restricted to explicitly authenticated tokens because of the inherent
security implications.

Obviously, different platforms will have different requirement regading:
* Whether long or short term remember me tokens are available?
* Whether explicitly authenticated tokens should be automatically demoted?
* What is available to each kind of token?

These are important security questions that need to be answered on a case-by-case basis.
