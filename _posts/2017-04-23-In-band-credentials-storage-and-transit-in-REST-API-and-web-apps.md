---
layout: post
title: "In band credentials storage and transit for REST APIs in the context of Web Apps (Part 2b)"
---

While writting the the second part of my [Authentatication and security for REST API in the
context of Web Apps](/2017/04/22/REST-APIs-authentication-and-security.html)
serie of posts, I discovered that [discussing the different alternatives to store credentials token](/2017/04/23/Credentials-storage-and-transit-in-REST-API-and-web-apps.html)
was more extensive than I previously thought.

This post focuses on storing and transmitting credentials in band. This means the
API is designed in such a way that the underlying Web App is aware and directly manages
the credential token (e.g., Web Storage). The other half of the discussion focuses on
[out-of-band credential storage](2017/04/23/Out-of-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html).

Most REST API libraries, frameworks, and essays in general, recommend using the
[Authentication Bearer](https://tools.ietf.org/html/rfc6750#section-2.1) HTTP mechanism.
Which is pretty much perfect from a REST point of view: it is stateless and
handled manually by the app (no browser/user-agent handle them automatically
the way they handle cookies, yet).

In any case, even if the tokens need to be manually managed by the app, it wouldn't be practical
to require the developer to do it independantly for each and every request. The credentials and authentication
management part MUST be segregated from the business logic part of the application's code. Both to avoid code duplication,
which is extremely error-prone and hard to maintain, and to reduce the exposition surface of the tokens.
Having a smaller amount of code dealing with credentials and token will mean requiring an attacker to analyse more
code before being able to exploit any potential XSS. Once this segregation is in place, the tokens would basically
become an out-of-band resource for the business logic part of the app.

Either way, an attacker could trivially either duplicate themself the authentication code or leverage the authentication
library of the attacked app. Therefore, none of these solutions will mitigate XSS vulnerabilities in any ways and they
MUST be addressed in other ways.

Since the token is available in-band to the Web App itself, an attacker would not need to trigger XTS or other
convulted attack to be able to steal the token.

However, REST and HATEOAS's goal is to make machine crawling of an
API easy, using only software without *a priori* knowledge of the API or service behind it.
Therefore an attacker does not need to steal and copy any token. Simply by leveraing
your own Web App's library, the attacker will be able to extract every single piece of information
from your API, automatically. Token theft is no so much of a bigger threat and it can be mitigated with
a properly designed API and appropriate IDS systems (either automatic or manual).

It is important to note, that Web storage APIs, although
[broadly implemented and available](http://caniuse.com/#feat=namevalue-storage),
still have some issues or unavailability depending on the browser configuration (e.g. private browsing).

For these reasons, some Web App and API will still require cookie based token management.
This can be mitigating by restraining the access to this feature in the API. And forcing tokens
to include how they should be used (as a Bearer manually managed token or via cookies).

There are several ways to store tokens in band. Each with their pros and cons.

#### An encapsulated Javascript variable

The biggest advantage to this solution is technology availability. Any browser able to handle an
SPA will be able to store a variable somewhere and send it back to the server.

On the cons side, this solution offers no permanent storage (no long term tokens can be effectively used).
It will also cut the user from well known and expected behavior like session forking (opening a link in a new tab or window).

By opening a link in a separate tab, the user will load a new instance of the Web app, which doesn't have any access to
the javascript run time environment of the previous app. This will require the user to login again.

In cases where the App is designed in such a way that this this behavior would create issues anyway,
this might be an acceptable solution for short term tokens. However, I don't see any design choice or workflow example
fitting this description and still being provided by a RESTful Web App and API (in particular,
one would need to sacrifice the stateless relationship with the server).

This solution SHOULD NOT be used unless the browser environment is such that session forking is impossible.

#### [Session storage](https://www.w3.org/TR/webstorage/#the-sessionstorage-attribute)

Session storage is to local storage, what session cookies are to permanent/time-based expiration cookies.

Session storage is basically a local storage bound to a session. This is not a permanent storage.
Hence, this is not appropriate for long term tokens.

However, session storage specification requires supporting session forking. There are still some use
cases the user wouldn't get their session storage back (implementor can test the feature using http://jsfiddle.net/moucdygg/2/).

Further broadening support of what is considering session forking or not will probably improve in the future.
Thus, making session storage more and more reliable as a acceptable short-term token storage. Moreover,
using a remember me token in a permanent storage (even a short term one), could mitigate these issues quite effectively.

Plus, since the storage is automatically cleared when the session is closed, this will reduce potentially higher value
(explicitly authenticated) short term tokens' exposure, even if just a little.

Therefore, when possible, session storage SHOULD be used as a short term token store.

#### [Credential Management](https://www.w3.org/TR/credential-management/)

The credential management specification is designed to allow broswers and user-agents to easily
and securely store and manage user credentials. Either explicitly or silently provided to a Web App.
They are not intended to be harder to use than standard Web storage, but offer better user controllability.

Short term tokens MAY be stored in Credential Management storage.

However, credentials are segregated by origin, but not by session. Thus, if allowing the user to
open multiple sessions at once (e.g., by creating multiple tabs or window not sharing session storage)
is a desirable feature, Credential Management storage SHOULD NOT be used to store short-term tokens.

On the other hand, Credential Managment storage is designed and extremely well-suited to host long-term tokens.
It SHOULD be used to store long term tokens. Unfortunately, at the time of writting this, Credential Management are
[very far from being widely supported](http://caniuse.com/#feat=credential-management). Other solutions will therefore
be required until they are widely available.

Whether to use it or not for primary credentials is outside the scope of the current discussion.

#### [Local storage](https://www.w3.org/TR/webstorage/)

Local storage is, currently, the only widely supported manually managed permanent storage available to Web Apps.
Local storage is safely segregated using the same-origin policy and offer cross-session durability.

However, given their durability, and the fact that there is way to expire data in it,
explicitly authenticated (high value) tokens MUST NOT be stored in local storage. If the support for 
session forking of the session storage is deemed inappropriate short term tokens MAY be stored in local storage.
However, in general, short term token are more appropriately stored in session storage and SHOULD NOT
be stored in local storage.

Long term tokens are more appropriately stored in secured credentials specific storages
(such as [Credential Management](https://www.w3.org/TR/credential-management/) storage).
However, until the implementation of such credential-specific secure storage is widely available,
local storage MAY be used to store long term tokens.
