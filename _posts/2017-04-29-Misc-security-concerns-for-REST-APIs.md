---
layout: post
title: "Miscellaneous security concerns in REST APIs (Part 6)"
---

This is the sixth (and last, but not least) part of my "[Authentatication and security for REST API in the
context of Web Apps](/2017/04/22/REST-APIs-authentication-and-security.html)"
series of posts. I hope I covered each steps of designing a RESTful authentication API. Although
security in the authentication mechanism is paramount. A secure authentication layer ensures no user will be impersonated.
It allows ensure no data will be stolen from the user, either because of phishing-like attack or because of tracking and
privacy-lacking tokens.

Here, we won't focus on anything in particular, but rather try to take a tour of general solutions for securing
or obfuscating a REST API and other issues.

Putting in place restrictions useful from a security standpoint without providing further information on the inner working
of a platform is often an important issue, and might sometime be counter-productive. For example, encrypting and validating
inputs can give rise to timing attacks. Another example could be providing a strict set of operation can actually expose
the inner working of the application simply by focusing on which operations are allowed and easy and which are forbidden or
complexe (either to execute or to reach). For these reasons, focusing on security will also require us to take a look at
broad API design considerations and suggestions.

### Restricting data manipulation

Many Web Apps or APIs restrict data manipulation to a narrow set of operations.
These operations usually take the form of a pre-determined set or sequence of HTML forms or REST calls.
Each with its own set of fields and allowed values.

Since REST architectures should be resource-based and not operation-based, this is usually a common pitfal while trying
to design a RESTful API.

This has the disadvantage of restricting greatly the potential UIs and UX implementations,
and confusing API-agnostic clients.
Moreover, carrying on these operations often requires an extensive set of ad-hoc error-prone checks and verifications.

A more resoure-centric way to go, would be to have a liberal resource schema, with very few mandatory fields (ideally none),
and a strict set of validation constraints (invariants).

Updating an existing resource SHOULD be done using either PUT or PATCH operations with as few
restrictions on the allowed modification set as possible.

Although PUT is the most RESTful HTTP operation to update a resource, it might be impractical to upload the whole
resource each time a slight modification is required.

PATCH's goal is to update the state of a resource.
This doesn't mean a partial upload or a partial put, this means updating a resource with a precise list of operations.
A media type with a well defined semantic MUST be used for PATCH operations (such as [JSON patch](http://jsonpatch.com/)
or [JSON merge patch](https://tools.ietf.org/html/rfc7396)).

Both PATCH and PUT operations SHOULD be safe-garded by using revalidating requests (i.e. using the If-Match header).

The set of read-write fields, the possible values for enum-based fields, and the pattern-matching restrictions
for each field SHOULD be specified in the API documentation for the resource.
This SHOULD represent accurately the current set of invariants for the resource and allow each client to present
its prefered UI to the user.

A resource MAY use several sets of invariants during its lifecycle. However, a change in the set of invariant SHOULD be
triggered by a POST operation. A change in the set of invariants will often be require when a resource's state
change implies additional side effects, such as sending an email, or interacting with an external service.
A POST operation makes such side effects more apparent.

To trigger these side effects correctly, the server will usually need to apply further validation to the resources,
such as requiring some fields and ensuring such and such field has a satisfying value.

Using POST operations only for side-effects and invariant transitions should reduce significantly the number
of complex and fragile operations.

Reducing the number of operations available and homogenizing them will reduce both the amount of code (therefore 
the possibility of bugs) and attack surface. Relying on static invariants (usually much easier to define and test)
rather than dynamic ones will also reduce the attack surface and reduce constraints on clients.

Creating a resource SHOULD be done using either of these solutions:
1. Sending a POST operation on a special resource creation endpoing for this type of resource.
2. Sending a PATCH operation to the listing endpoint of the resource. The semantic of this PATCH operation would be to
   modify the list by adding an element to it (the new resource).

In any case, this operation SHOULD take the necessary minimalist information to initialize the resource
(ideally none) as input.

Deletion of a resource SHOULD be done with a DELETE operation. It also (in addition or alternatively) MAY be done by
sending a PATCH request on the listing endpoint removing the appropriate index from the list.

For creation and deletion, different API will probably choose different approaches for different resources.
For example, a POST/DELETE scheme might be better for resources without a global canonical listing endpoint,
while the listing PATCH-only approach could be better for subresources and list-valued properties.

If a more integrated user experience is sought after, it should be taken care of in the Web App, not enforced by the API.
This also means that changing the whole user experience should only require modifications in
the Web App and not in the API.

From an obfuscation standpoint, this is similar to using non-consecutive or obfuscated IDs,
although on a much broader scale.
It will make listing the intended or potential operations on the resources so abstract,
the API will give almost no information on how to manipulate the resources maliciously. It will also be harder for an
attacker to determine which potential modification would be meaningful and which ones would be trivial in nature.
A very constrained API, on the other hand, gives so much information on its
intentions, it may give a lot of information on its inner working.

### 404 vs 403

Many applications today use 404 Not Found to mask the existence of potential resources the user isn't granted access to.
As opposed to 403 Forbidden, 404 are the de facto standard for refusing access.

However, this is often done is a way that violate the concept of manipulating resources, but without
actually hidding information.

For example, let's say a user has read/write access to a resource A, but read-only access to a resource B. Typically,
current APIs would advertise operations (POST,PATCH,PUT) to update resource A, but not resource B.

Most of the time, the API will even document these operations on a global level. Therefore, even if the user
has access to no resources of this type, they will be aware of the existence of these operations.

In this context returning a 404s for a write operation on B is utterly misleading:
* The client already knows the resource exists, or can know it trivially by sending a GET request.
* A poorly written or with heavily restricted resources client, by getting a 404, could assume that
  the resource has been deleted and invalidate its local caches. Thus generating more load on the server.
* The API advertise inconsistent documentation by, on one hand, advertising the existence of the operation for every
  resources of this type, and, on the other hand, advertising the non-existence of the same operation on some resources.

This behavior comes with absolutely no value and carries the potential cost of confusing clients and user agents.

However, if the user doesn't have access, at all to a resource C, returning a 404 for this resource C,
for any operation on it, will effectively hide the very existence of this resource.

For these reasons, returning 404 for a resource is acceptable only if the user would have no way of determining
whether the resource exists. Since RESTful API are designed to be automatically discoverable, any information available 
on a given resource should be found trivially by automated tools. In this context, restricting 404s to completely hidden
resources is not increasing the amount of information or the simplicity of finding such information.

In particular:
* A POST, PATCH or PUT operation on a read-only known resource SHOULD return a 405 Method Not Supported.
* A method never advertised to the user for the appropriate type of resource SHOULD return a 405 Method Not Supported.
* An invalid POST or PATCH operation on a known resource SHOULD return a 400 Bad Request.

Other HTTP Status error codes should be returned appropriately. Most error codes will be triggered
at the explicit request of the client (e.g., 406, 412, 415 or 417) and replacing them by 404s or generic 400s will
only make integration with API-agnostic clients more difficult.

The only error code that would provide information that would not be available otherwise is 500 for an
unexpected error. This one COULD be be replaced by a 400 error code in production, in order to mask server crashes.

However, special care must be taken to avoid timing attacks: If some 400 are returned early during the request
processing and other much later, it will be easy to segregate the ones triggered during the early request validation step
from the ones triggered by a crash later on.

### Obfuscating URLs

Many industry-level web applications take great care to obfuscate their IRLs. Thus making more difficult for a potential
attacker or automated script to SPAM the API in the hope to find a flaw.

Many current APIs, usually based on JSON or XML, and presented as RESTful,
try to constrain using (almost) human-readable IRLs.
An example of such an IRL could be: `/blog/128/comment/48`.

From an obfuscation standpoint, there are two issues:
1. It gives a very easy way to ascertain the amount of activity and data in your service
2. It allows anyone to list your resources by simply brute-forcing the ID part.

The first point can actually make you a more worthy target or allow an attacker to tracker users activity by
monitoring the id publicized.

It also renders the common practice to replace 403 by 404 errors completely useless. Any ID lower than a known one,
can be assumed to be existent and forbidden.

The second point will advertise a lower cost target. Since an attacker can have a pretty good idea of how many
IRLs they need to attack.

#### Resources ID randomization

A easy solution for this is to generate randomized IDs.
By presenting arbitrarily larges and non-monotones IDs,
any attacker will immediately notice that they are randomized. It will basically signal two things:
* No information whatsoever on your website activity can be obtained by looking at the IDs,
* The 404 answer for both the ID=10 and ID=87455589 resource are most probably because the resources actually do not exist.

An attacker will always prefer a target with an attack surface easy to determine rather than a potentially much broader one.
This is the reason why these facts needs to be publicized upfront.

This would mitigate effectively both aformentioned obfuscation issues.

Special care must be taken to ensure timing attacks are not possible. A request for a non-existent resource will typically
be recognized more quickly (as soon as the resource lookup abort) than a request for a forbidden resource (which needs to
go through the authorization verification process).
This difference can be used by an attacker to determine which resources actually exist or not.
Requests on an actually non-existent resources MUST be slowed down to the level of a forbidden resource.
This will have the side effect of Rate Limiting resource probing, for no additional cost.
In connection intensive environments, the waiting period could be delegated to a separate HTTP endpoint.
This separate endpoint could be delegated the reponsibility for TLS endpoint, load balancer, web application firewalling,
IDS system, caching, and resource composition or normalization and other. These are all responsibilities that do not
require to be hosted on the main applicative server.

However, for some applications, the simple fact that URL parameters exist can provide a trove of data, which brings us to
our next discussion.

### Complete URL obfuscation

Let's take the example of a big and complexe banking application.
They have many resources:
* Accounts
* User profiles
* Secure messages
* Cheque pictures
* Transfers
* Product order
* Shares market operations
* and so on...

Simply having `/account/98234798/stock/234987493` already can expose the fact that account 98234798, whatever it is, is
a market account, not a chequing or a savings account. This can open or close entire families of attack on the account.

Since REST is actually Hypermedia driven, there is no actual restriction on how should the URL be written.

In such an extremely hostile environment, URLs' path SHOULD be completely encrypted. For example,
using AES256 or a public/private key signing algorithm and using an url-safe base64 encoding to
produce the public path.

Such an encryption layer could be abstracted in an URL management library, which would allow developers to disable it
during the development phase, for ease of access and debugging and seamlessly enforce it in production.

Having a strict underlying path pattern will allow URL padding to ensure the URLs' size cannot be determined.
Forbidding URL templating will ensure no additional information is leaked from the URL.

In this case, additional care MUST be taken to ensure resource access is checked as soon as possible
and in a consistent manner. Otherwise, timing attacks would become possible (See previous point).

This solution alone is obviously useless if every media types and operation is advertised right of
the bat in the API documentation.
To prevent such situation, APIs SHOULD advertise a very limited portion of themselves in their global documentation.

For example, a user connecting to our banking application would be
provided with the following endpoints (after authentication):
* An endpoint to list their accounts
* An endpoint to access their own profile
* An endpoint to list their secure messages

And that should be it. The documentation SHOULD use non-dereferenceable or generic IRLs to identify
types and other identifying parts of the API. This will ensure no further information on the URLs is provided.

When an endpoint is accessed, its result will include the documentation for further operations and access to
other resources.

This solution will make it much harder to generate a complete map of the application or API.
A user will only ever be advertised the operations and resources they have access to and are using.

Let's say a user A doesn't have any stock-market account, they will never know what resources stock-market accounts offer
or what operations they support.
This could allow the API provider to seamlessly restrict access to the API based on a pricing plan or paid features, while
retaining essential REST characteristics. Including resources manipulation, API-agnostic clients and Hypermedia driven API.

These are extreme cases and SHOULD NOT be implemented without careful considerations. This will very often render several
URL-based tools useless and may make developping a Web App supporting the API much more difficult.

It is important to note as well that these encrypted URLs should only be the API's endpoints, not the URLs made visible
to the user by the Web App (usually a SPA). These URLs will usually not require such a level of obfuscation.

A scheme providing enough information to identify both the resource and the component that need to be activated upon
reloading the Web App would be enough.
Such a scheme could, for example, reference the server resources using indexes within the users' list of available
resources or other meaningless path to the resource. These URLs SHOULD NOT contain any part of the resources' actual ID
(its endpoint within the API), but only provide a way to navigate the resource graph itself from a well known entrypoint.
This way even if a user copy-paste the URL they see to someone else, this will provide no information whatsoever on the
API and the underlying datastore.

This also allow the Web App to present user friendly URLs to the user while using opaque URLs when communicating with the
server.
