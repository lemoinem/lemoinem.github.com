---
layout: post
title: "Miscellaneous security concerns in REST APIs (Part 6)"
---

This is the sixth and last (but not least) part of
my
[Authentication and security for REST API in the context of Web Apps][intro]
series of posts. We covered each steps of designing
and [securing a RESTful authentication API][part-5].

Security in the authentication mechanism is paramount. A secure authentication
layer ensures no user will be (easily) impersonated. It also ensures no data
will be stolen from the user, either because of [phishing attacks][phishing]
(criteria 8) or because of [tracking and privacy-lacking][privacy] tokens
(criteria 10).

Here, we won't focus on anything related to authentication in particular, but
rather try to take a tour of general solutions for securing or obfuscating
a [RESTful][rest] API and other security issues.

Securing a Web App is often an uphill battle. Plugging one hole often means
opening another. For example:
* encrypting and validating inputs can give rise
  to [timing attacks][timing-attack],
* preventing [phishing attacks][phishing] by using anti-phishing proof, might
  open the door for [account enumeration attacks][account-enumeration],
* hiding the authentication token out-of-band to
  prevent [token theft][session-hijacking] will open the door to [CSRF][csrf]
  attacks,
* providing a strict set of operation can allow an attacker to learn the the
  inner working of the application simply by comparing which operations are
  allowed and easy to do and which are forbidden or complex (either to execute
  or to reach)...

Focusing on security will also require us to take a look at broader API design
considerations and ideas.

### Restricting data manipulation

Many Web Apps or APIs restrict data manipulation to a very narrow set of
operations. These operations usually take the form of a pre-determined set or
sequence of HTML forms or API calls. Each with its own set of fields and allowed
values.

Since [REST][rest] architectures should be resource-based and not
operation-based, this is usually a common pitfall while trying to design a
RESTful API.

This has the disadvantages of restricting greatly the potential UIs and UX
implementations, and confusing API-agnostic clients. Moreover, carrying on these
operations often requires an extensive set of ad-hoc error-prone checks and
verifications.

### Resource-centric approach to data manipulation

A more resource-centric and [RESTful][rest] approach, would be to produce a
liberal resource schema, with very few mandatory fields (ideally none), and a
strict set of validation constraints (invariants).

Updating an existing resource SHOULD be done using either `PUT` or `PATCH`
operations, with as few restrictions on the allowed modifications set as
possible.

Although `PUT` is the most [RESTful][rest] HTTP operation to update a resource,
it might be impractical to upload the whole resource each time a slight
modification is required.

`PATCH`'s goal is to update the state of a resource. This doesn't mean a partial
upload or a partial `PUT`. It means updating a resource using a precise list of
operations. A media type with a well defined semantic MUST be used for `PATCH`
operations (such as [JSON patch][json-patch]
or [JSON merge patch][json-merge-patch]).

`POST`, `PATCH` and `PUT` operations SHOULD be safe-guarded by
using [precondition headers][precondition-header] (e.g., `If-Match`).

The set of read-write fields, the possible values for enum-based fields, and the
pattern-matching restrictions for each field SHOULD be specified in the API
Documentation for the resource. This SHOULD represent accurately the current set
of invariants for the resource and allow each client to present its preferred UI
to the user.

A resource MAY use several sets of invariants during its life-cycle. However, a
change in the set of invariant SHOULD be triggered by a `POST` operation. A
change in the set of invariants will often be required when a resource's state
change implies additional side effects, such as sending an email, or interacting
with an external service.  A `POST` operation makes such side effects more
apparent.

To trigger these side effects correctly, the server will usually need to apply
further validation to the resources, such as requiring some fields and ensuring
such and such field has a satisfying value.

Using `POST` operations only for side-effects and invariant transitions should
reduce significantly the number of complex and fragile operations.

Reducing the number of operations available and homogenizing them will reduce
both the amount of code (therefore the possibility of bugs) and attack
surface. Relying on static invariants (usually much easier to define and test)
rather than dynamic ones will also reduce the attack surface and reduce
constraints on clients.

Creating a resource SHOULD be done using either of these solutions:
1. Sending a `POST` operation on a special resource creation endpoint for this
   type of resource.
2. Sending a `PATCH` operation to the listing endpoint of the resource. The
   semantic of this `PATCH` operation would be to modify the list by adding an
   element to it (the new resource).

In any case, this operation SHOULD take the necessary minimalist information to
initialize the resource (ideally none) as input.

Deletion of a resource SHOULD be done with a `DELETE` operation. It also (in
addition or alternatively) MAY be done by sending a `PATCH` request on the
listing endpoint removing the appropriate index from the list.

For creation and deletion, different API will probably choose different
approaches for different resources.  For example, a `POST`/`DELETE` scheme might
be better for resources without a global canonical listing endpoint, while the
listing `PATCH`-only approach could be better for sub-resources and list-valued
properties.

If a more integrated user experience is sought after, it should be taken care of
in the Web App, not enforced by the API. This also means that changing the whole
user experience should only require modifications in the Web App and not in the
API.

From an obfuscation standpoint, this is similar to using non-consecutive or
obfuscated IDs, although on a much broader scale. It will make listing the
intended or potential operations on the resources so abstract, the API will give
almost no information on how to manipulate the resources maliciously. It will
also be harder for an attacker to determine which potential modification would
be meaningful and which ones would be trivial in nature. A very constrained API,
on the other hand, gives so much information on its intentions, it may give a
lot of information on its inner working.

### 404 vs 403

Many applications nowadays use "404 Not Found" to mask the existence of
resources the user isn't granted access to. As opposed to "403 Forbidden", "404
Not Found" are the de facto standard for refusing access in industry-grade Web
Apps.

However, this is often done is a way that violate the concept
of [manipulating resources][rest], but without actually hiding information.

For example, let's say a user has read/write access to a resource `A`, but
read-only access to a resource `B`. Typically, current APIs or Web Apps would
advertise read/write operations (`POST`,`PATCH`,`PUT`) to update resource A, but
not resource B.

Most of the time, the API will even document these read/write operations on a
global level, in the API Documentation for the API's entry-point. Therefore,
even if the user has access to no resources of this type, they will be aware of
the existence of these operations.

In this context returning a "404 Not Found" for a write operation on B is
utterly misleading:
* The client already knows the resource exists, or can know it trivially by
  sending a `GET` request. Plus the information is inconsistent based on the
  HTTP Verb. In a resource-centric approach, it should not be.
* A poorly written or with heavily restricted resources client, by getting a
  404, could assume that the resource has been deleted and discard its local
  cache for the resource. Thus generating more load on the server.
* The API ends up advertising inconsistent documentation: On one hand, the
  existence of the operations for every resources of this type is presented,
  and, on the other hand, advertising the non-existence of the same operation on
  some resources.

This behavior comes with absolutely no value and carries the potential cost of
confusing clients and user agents.

However, if the user doesn't have access, at all to a resource C, returning a
"404 Not Found" for this resource C, for any operation on it, will effectively
hide the very existence of this resource. This is an example of a valid use of
"404 Not Found" to mask a resource.

For these reasons, the server MAY return a "404 Not Found" for an existing
resource only if the user has no other way of determining whether the resource
exists. Since [RESTful][rest] APIs are designed to
be [automatically discoverable][hateoas], any information available on a given
resource will be found trivially by automated tools. In this context,
restricting "404 Not Found"s responses to thoroughly hidden resources is not
increasing the amount of information or the simplicity of finding such
information.

Other HTTP Status error codes should be returned appropriately. Most error codes
will be triggered at the explicit request of the client (e.g., 406, 412, 415 or
417) and replacing them by "404 Not Found" or generic "400 Bad Request" will
only make integration with API-agnostic clients more difficult.

In particular:
* A `POST`, `PATCH` or `PUT` operation on a read-only known-to-exist resource
  SHOULD return a "405 Method Not Supported" response.
* A method never advertised to the user for a resource SHOULD return a "405
  Method Not Supported" response.
* An invalid `POST` or `PATCH` operation on a known resource SHOULD return a
  "400 Bad Request".

The only error code that would provide information that would not be available
otherwise is "500 Internal Server Error" for an unexpected error. This one MAY
be be replaced by a "400 Bad Request" error code in production, in order to mask
server crashes.

However, special care must be taken to avoid timing attacks: If some "400 Bad
Request"s are returned early during the request processing and other much later,
it will be easy to segregate the ones triggered during the early request
validation step from the ones triggered by a crash later on.

### Obfuscating IRIs

Many industry-level web applications take great care to obfuscate their
IRIs. Thus making more difficult for a potential attacker or automated script to
spam the API in the hope to find a flaw.

Many current APIs, usually based on [JSON][json] or [XML][xml], and presented
as [RESTful][rest], try to constrain using (almost) human-readable IRIs.  An
example of such an IRI could be: `/blog/128/comment/48`.

Although these IRIs are great from a [SEO][seo] and human point-of-view, they
give rise to two issues:
1. They give a very easy way to ascertain the amount of activity and data in
   your service. (A new resource being assigned an ID very close to the last
   known one means the website doesn't have much activity. A new resource being
   assigned a high ID means there are many resources hosted on the server.)
2. It allows anyone to list your resources by
   simply [brute-forcing the ID part][forced-browsing].

The first point can actually make you a more worthy target or allow an attacker
to track other users activity.

It also renders the common practice to replace "403 Forbidden" by "404 Not
Found" completely useless. Any ID lower than a known one can be assumed to
exist.

The second point will advertise a lower cost target. Since an attacker can have
a pretty good idea of how many IRIs they need to attack.

#### Resources ID randomization

An easy solution for this is to generate randomized IDs. By presenting
arbitrarily larges and non-monotones IDs, any attacker will immediately notice
that they are randomized. It will basically signal two things:
* No information whatsoever on your website activity can be obtained by looking
  at the IDs,
* The 404 answer for both the ID=10 and ID=87455589 resource are most probably
  because the resources actually do not exist.

An attacker will always prefer a target with an attack surface easy to determine
rather than a potentially much broader one. This is the reason why these facts
need to be publicized upfront.

This would mitigate effectively both issues previously mentioned, while still
offering the same human readable and [SEO][seo] features.

Special care must be taken to ensure [timing attacks][timing-attack] are not
possible. A request for a non-existent resource will typically be recognized
more quickly (as soon as the resource lookup aborts) than a request for a
forbidden resource (which needs to go through the authorization verification
process). This difference can be used by an attacker to determine which
resources actually exist or not.

Requests on an actually non-existent resources MUST be slowed down to the level
of a forbidden resource. This will have the side effect of Rate Limiting
resource probing, for no additional cost. In connection intensive environments,
the waiting period could be delegated to a separate HTTP server, set up as a
reverse proxy. This separate server could be delegated the responsibility for TLS
endpoint, load balancing, web application firewall, IDS system, caching,
resource composition or normalization, and others. These are all
responsibilities that do not require to be hosted on the main applicative
server.

However, for some applications, the simple fact that IRI parameters exist can
provide a trove of data, which brings us to our next discussion.

### Complete IRI obfuscation

Let's take the example of a big and complex banking application. They have many
resources:
* Accounts
* User profiles
* Secure messages
* Check pictures
* Transfers
* Product order
* Shares market operations
* and so on...

Simply having `/account/98234798/stock/234987493` already can expose the fact
that account 98234798, whatever it is, is a market account, not a chequing or a
savings account. This can trigger an attacker to try specific strategies for
specific specific accounts.

Since [REST][rest] is actually [Hypermedia driven][hateoas], there is no actual
restriction on how should the IRIs be written.

In such an extremely hostile environment, IRIs' path SHOULD be completely
encrypted. For example, using AES256 or a public/private key algorithm and using
an IRI-safe base64 encoding to produce the public path.

Such an encryption layer could be abstracted in an IRI management library, or
enforce by a reverse proxy, which would allow developers to disable it during
development, for ease of access and debugging.

Having a strict underlying path pattern will allow IRI padding to ensure the
IRIs' size cannot be determined. Forbidding IRI templating will ensure no
additional information is leaked from the IRI and effectively increase the cost
of [resource enumeration][forced-browsing] to a prohibitive value.

In this case, additional care MUST be taken to ensure resource access is checked
as soon as possible and in a consistent
manner. Otherwise, [timing attacks][timing-attack] would become possible (See
previous point).

This solution alone is obviously useless if every media types and operation is
advertised in the API Documentation for the API's entry point. To prevent such
situation, APIs with complete IRI obfuscation SHOULD advertise a very limited
portion of themselves in their global documentation.

For example, a user connecting to our banking application would be provided with
the following endpoints (after authentication):
* An endpoint to list their accounts
* An endpoint to access their own profile
* An endpoint to list their secure messages

And that should be it. The documentation SHOULD use non-dereferenceable or
generic IRIs to identify types and other identifying parts of the API. This will
ensure no further information on the IRIs is provided.

When an resource is accessed, its result will include the documentation for
further operations and access to other resources.

This solution will make it much harder to generate a complete map of the
application or API. A user will only ever be advertised the operations and
resources they have access to and are using.

Let's say a user A doesn't have any stock-market account, they will never know
what resources stock-market accounts offer or what operations they support.
This could allow the API provider to seamlessly restrict access to the API based
on a pricing plan or paid features, while retaining essential [REST][rest]
characteristics. Including resources manipulation, API-agnostic clients
and [Hypermedia driven][hateoas] API.

These are extreme cases and SHOULD NOT be implemented without careful
considerations. This will very often render several IRI-based tools useless and
may make development or debugging an integrated Web App supporting the API much
more difficult.

It is important to note as well that these encrypted IRIs should only be the
API's endpoints, not the IRIs made visible to the user by an [SPA][spa]. The
later will usually not require such a level of obfuscation.

A scheme providing enough information to identify both the resource and the
component that need to be activated upon reloading the Web App would be enough.
Such a scheme could, for example, reference the server resources using indexes
within the users' list of available resources, or other paths meaningless for an
attacker without access to the resources themselves. These IRIs SHOULD NOT
contain any part of the resources' actual ID (its endpoint within the API), but
only provide a way to navigate the resource graph itself from a well known
entry-point. This way even if a user copy-paste the IRI they see to someone else,
this will provide no information whatsoever on the API and the underlying
datastore.

This also allow the Web App to present user friendly IRIs to the user while
using opaque IRIs when communicating with the server.

[intro]:  /2017/04/22/REST-APIs-authentication-and-security.html "Authentication and security for REST API in the context of Web Apps"
[part-5]: /2017/04/29/Securing-a-RESTful-authentication-API.html "Security of a RESTful authentication API"

[rest]:    https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm "Architectural Styles and the Design of Network-based Software Architectures"
[hateoas]: https://en.wikipedia.org/wiki/HATEOAS                       "Hypermedia As The Engine Of Application State"

[spa]: https://en.wikipedia.org/wiki/Single-page_application    "Single-page application"
[seo]: https://en.wikipedia.org/wiki/Search_engine_optimization "Search engine optimization"

[csrf]:                https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)                                          "Cross-Site Request Forgery (CSRF)"
[phishing]:            https://www.owasp.org/index.php/Phishing                                                                   "Phishing"
[account-enumeration]: https://www.owasp.org/index.php/Testing_for_Account_Enumeration_and_Guessable_User_Account_(OTG-IDENT-004) "Account Enumeration"
[session-hijacking]:   https://www.owasp.org/index.php/Session_hijacking_attack                                                   "Session hijacking attack"
[forced-browsing]:     https://www.owasp.org/index.php/Forced_browsing                                                            "Forced browsing"

[precondition-header]: https://tools.ietf.org/html/rfc7232#section-3 "HTTP/1.1: Conditional Requests - Precondition Header Fields"

[timing-attack]: https://en.wikipedia.org/wiki/Timing_attack "Timing attack"

[privacy]: https://en.wikipedia.org/wiki/Internet_privacy "Internet privacy"

[json]: http://www.json.org/              "Introducing JSON"
[xml]:  https://en.wikipedia.org/wiki/XML "Extensible Markup Language"

[json-patch]:       http://jsonpatch.com/               "What is JSON patch?"
[json-merge-patch]: https://tools.ietf.org/html/rfc7396 "JSON Merge Patch"
