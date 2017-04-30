---
layout: post
title: "Authentication and security for REST API in the context of Web Apps (Intro)"
---

I am, finally, starting to look at developing REST APIs as the backend of a Web
Apps. I'm certainly not the first and the community as already been developing
great tools and technologies for this.

In particular, given my background with PHP and Symfony, I've been looking at:
* [JSON-LD][json-ld],
* [JWT][jwt],
* [HYDRA][hydra],
* [API Platform][api-platform]

For anyone, who haven't read [Roy T. Fielding's thesis][rest], I would highly
recommend it. As much because it is THE document birthing the REST architectural
style, as because it offers extremely interesting insight on Web and HTTP
technologies.

However, there is a subject on which I haven't found much resources, which is
"_How to design a RESTful and secure authentication API?_" and the topic that
obviously follow it: "_How to integrate this with a RESTful API?_".  Most
platforms and tools focus on the API itself, putting security aside as a
secondary concern. Even API Platform, which is a very promising platform,
extremely developer friendly, and with an active community, doesn't even try to
offer [decent default security settings][api-platform:#109] yet.

In the next posts, I will try to provide my thoughts and ideas on the subject.

My main concerns or criteria are (in no particular order, but numbered for easy
reference):
1.  [RESTfulness][rest]: as much as possible,
    including [statelessness][stateless] and the [HATEOAS][hateoas] principle;
2.  Genericity: although the discussion will
    use [Linked data][linked-data], [JSON-LD][json-ld] and [Hydra][hydra], the
    solutions proposed need to be easily transposed to other technologies;
3.  Cookies: they are very hard to avoid when discussing current Web Apps;
4.  Different levels of request authentication: anonymously, remembered &
    explicitly authenticated requests;
5.  Multiple schemes of user authentication:
    * password-based,
    * token-based,
    * MFA,
    * external trustee (OAuth/OpenID);
6.  [CSRF][csrf] Vulnerabilities (including [Login CSRF][login-csrf]);
7.  [XSS][xss] vulnerabilities;
8.  [Phishing][phishing] attacks;
9.  [Account enumeration][account-enumeration] vulnerabilities (and other forms
    of resource enumeration);
10. [User tracking][privacy] by a third party;
11. [Token theft][session-hijacking].

I will try to address and discuss these concerns and offer better insight on
what should be important and why in the following posts:

1. [Token's format (Credentials)](/2017/04/23/credentials-for-REST-API.html)
2. [Credentials storage and transit](/2017/04/23/Credentials-storage-and-transit-in-REST-API-and-web-apps.html):
    1. [Out-of-band credentials storage and transit](/2017/04/23/Out-of-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html)
    2. [In band credentials storage and transit](/2017/04/23/In-band-credentials-storage-and-transit-in-REST-API-and-web-apps.html)
3. [RESTful authentication API](/2017/04/23/REST-authentication-API-for-Web-App.html)
4. [Modularity of RESTful authentication API](/2017/04/27/Modular-REST-authentication-API-for-Web-App.html)
5. [Security of a RESTful authentication API](/2017/04/29/Securing-a-RESTful-authentication-API.html)
6. [Other security consideration for RESTful APIs](/2017/04/29/Misc-security-concerns-for-REST-APIs.html)

[rest]:      https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm "Architectural Styles and the Design of Network-based Software Architectures"
[stateless]: https://en.wikipedia.org/wiki/Stateless_protocol            "Stateless protocol"
[hateoas]:   https://en.wikipedia.org/wiki/HATEOAS                       "Hypermedia As The Engine Of Application State"

[linked-data]:  http://linkeddata.org/                 "Connect Distributed Data across the Web"
[json-ld]:      http://json-ld.org/                    "JSON for Linking Data"
[jwt]:          https://jwt.io/                        "JSON Web Tokens"
[hydra]:        http://www.markus-lanthaler.com/hydra/ "Hypermedia-Driven Web APIs"
[api-platform]: https://api-platform.com/              "PHP framework to build modern web APIs"

[api-platform:#109]: https://github.com/api-platform/api-platform/issues/109 "API Platform issue: [RFC] Come with security out of the box"

[csrf]:                https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)                                          "Cross-Site Request Forgery (CSRF)"
[xss]:                 https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)                                                 "Cross-site Scripting (XSS) "
[phishing]:            https://www.owasp.org/index.php/Phishing                                                                   "Phishing"
[account-enumeration]: https://www.owasp.org/index.php/Testing_for_Account_Enumeration_and_Guessable_User_Account_(OTG-IDENT-004) "Account Enumeration"
[session-hijacking]:   https://www.owasp.org/index.php/Session_hijacking_attack                                                   "Session hijacking attack"

[login-csrf]: http://www.adambarth.com/papers/2008/barth-jackson-mitchell-b.pdf "PDF: Robust Defenses for Cross-Site Request Forgery"

[privacy]: https://en.wikipedia.org/wiki/Internet_privacy "Internet privacy"
