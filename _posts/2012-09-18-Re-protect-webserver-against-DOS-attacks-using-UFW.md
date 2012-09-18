---
layout: post
title: "Re: Protect Webserver against DOS attacks using UFW"
---

This is my answer to a friend post [Protect Webserver against DOS attacks using
UFW](http://blog.lavoie.sl/2012/09/protect-webserver-against-dos-attacks.html). I
intended to post it as a comment, however, it looks like I was a bit carried up
and my answer was "a tad too long" for his comment system. *&#91;edit&#93;You
will find his own answer to my post there too.&#91;/edit&#93;*

Quite interesting article. Although the goal is important and utterly relevant,
considering the number of automatic bots now creeping around in our virtual
universe, I'm a bit afraid by the small thresholds. Imho, there is a very
important trick to get used to when designing protection system: You have to
make sure your own protection isn't going to get in your way and either protect
you from legitimate user scenario, or generate such a load that will make it
actually easier for you attacker to DOS your service/server.

I have several concern regarding your strategy that may fall within this scope:

- Human behavior expected: How do you manage visits by legitimate bots (for
  example: Search Engines) which are usually way faster than a human visitor and
  may (will) trigger your protections? Results: No more indexation by Google &
  Friends.

- Human behavior expected: What if your server also provide Web API and
  Web-services used by remote scripts or other services? Although this is
  obviously outside the scope of your article, I still think it is interesting
  to keep that in mind. This seems to be a more and more common case (the
  simplest example being dynamical parts of your website managed using
  repetitive AJAX)

- Single human behavior expected per IP or Class C: What if a company
  (relatively big) use NATing with a single or a small set of egress IP(s)? You
  may end up with a relatively heavy legitimate load from a single IP that you
  will shutdown.

- Class C restriction: How do you manage DDOS (from several Class Cs) at the
  same time, looks like your rules are going to ban whole unrelated class Cs?
  Which, if the attack is well distributed, is still going to alienate a good
  part of the tubes. Moreover, 1 Class C = 1 Company is not a reliable
  hypothesis anymore. It will be quite unusual for anybody except big old
  companies to own a whole Class C. Most companies will have a /27 or even
  /29. And most people have a single IP address provided by their ISP. You end
  up opening a door to indirect targeted DOS (using a few stepping
  stones/zombies, easier to infect, in the same Class C as a more secure
  individual/company one would like to lock out from your site).

- Human behavior expected by server: Your thresholds seems to expected the
  behavior of one human on ONE site on your server. I think you're speaking of
  webhosting here, with a potential of dozens of websites on the same
  server. The connection pattern is going to drive your firewall in lock out
  frenzy if one person/company browses several of these websites...

Moreover, there are a few issues that are not considered (Connection starvation,
for example) by your setup.

I don't think having conservative limits regarding your firewall is going to do
you any good if you start having a reasonably heavy legitimate load on your
server. Moreover, having unexpectedly complex firewall rules may still trigger
high CPU load, which would be the new target of your attacker once identified as
such: providing a distributed traffic always close to be rejected but not quite
matching the rule will generate CPU load both from the website and the firewall
which will probably make it even easier to crash your website.

---

I think a better way to go, would be first to get rid of any application-wide
vulnerabilities. I'm not speaking of SQL injections or XSS, which a functional
vulnerabilities but will rarely end up as a DOS strategy, but connection
starvation or the ability to trigger extreme load with one single query (poor
man search engines requests are usually a good example of such queries). If
someone seems to be abusing these, a nice temporary "sorry, 503" or "302" to
another server ban will most of the time be enough (and a captcha to unlock it
could be neat).

Then, design simple but effective liberal rules for your firewall. For example
extreme connections rates (such as several dozens connections from the same IP
within 2 or 3 seconds). These rules should be enabled if the CPU load (or
whatever your bottleneck is) rises above some high threshold. A simple
monitoring solution, such as Monit could do that job and send you an e-mail or
another alert at the same time so you know what's happening and can take direct,
mitigation efficient, actions. For example, a manual analysis of the logs could
give you a list of IPs to ban directly in the firewall for the next X hours or
so. Which will be highly effective, without sucking on your valuable CPU
resources.

I hope this may help. I think this system is still simple yet efficient
enough. You already ought to have some kind of monitoring solution, adding a few
tests and triggers should be a big deal and you will know first hand what's
going on with your server.
