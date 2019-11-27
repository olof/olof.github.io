---
layout: post
title: "DoH over Tor for anonymous, confidential DNS resolution"
categories: dns, privacy, tor, perl, dohot, dohotcd, doh
---
There's ongoing work on enabling DNS tunneling over HTTPS ([RFC
8484][rfc/8484], DNS-over-HTTPS (DoH)) for the purpose of getting
rid of passive observers seeing your query patterns by encrypting
the queries. My opinion: the network operator does not have a
"right" to see my data, regardless of protocol. I will always
reserve the right to obscure my traffic with technical means if I
feel like it.

Ok, so that makes me a proponent of DoH? Yes, I guess so! Some
people worry about this enabling a centralization of DNS: while I
agree that centralization of network services is a real worry,
*DoH* is not the change that makes name resolution centralized.
We already have the popular public resolvers from
Google/Cloudflare/Cisco/etc. We already have a centralization of
hosting resources in AWS/Google/Azure/etc.

The thing that bothers me the most about the rollout of DoH is
not the transport protocol, or how it affects "network
observability"; it is how it moves name resolution from system
configuration to the applications. It's not a bother in the sense
that "mozilla is evil", but only a technical reservation. I see
why Mozilla opted to roll it out the way they did in Firefox:
they control the browser, they don't control `/etc/resolv.conf`.

But as a self confessed DoH proponent, I can make my resolv.conf
play the HTTPS game by setting up a stub proxy, similar to [the
one implemented by Facebook][py/doh-proxy]. This has two
benefits: the main one being that all name resolution on the
systems uses DoH, without having to teach this to each individual
application. The second one is that our simple http client does
not have any access to any browser data stores or javascript
engines which limits the consequences of exploits.

But this means having to use a third party DoH provider or set
one up myself. Deploying one myself has severe privacy
implications: if I'm the only one using it, the query patterns
will still be easily observable by the network operator of the
DoH server.  For this reason, using a public DoH provider can
actually be said to have privacy benefits because your queries
are mixed with the queries from other users of that provider. But
obviously, this goes both ways. The DoH operator, with whom I did
not have a relationship before, will now see all my query data.

If you already use a public resolver like 8.8.8.8 or 1.1.1.1,
switching to DoH should be a no-brainer. The only consequence is
to remove the ability for network operators to monitor your
queries en route. But it can still be mitigated: My idea to solve
this is to make that communication go through Tor: DoH provides
the confidentiality and Tor provides the anonymity.  Turns out,
Cloudflare hackers also [thought about this][cf/tor]!

> the exceptionally privacy-conscious folks might not want to
> reveal their IP address to the resolver at all, and we respect
> that. This is why we are launching a Tor onion service for our
> resolver at
> dns4torpnlfs2ifuz2s2yf3fc7rdmsbhm6rw75euj35pac6ap25zgqad.onion
> and accessible via tor.cloudflare-dns.com.

But sadly this Cloudflare Onion service is no longer online. Had
this been online, things would have been a bit simpler. But I
still want this anonymity from the DNS provider, so what I will
do is somehow proxy the DoH traffic over Tor myself. To that end,
I wrote a small stub resolver, [dohotcd][dohotcd] (*DoHoT client
daemon*), that listens ons 53/udp and forwards the query over
https. The https request is proxied using privoxy and tor.
(Privoxy is in the mix because my HTTP client library doesn't
support socks proxies.) I configure my regular resolver
([unbound][unbound]) to forward all queries to this stub. Reasons
for still using my regular non-stub resolver:

* Caching! You *don't* want to forward every query over and over
  to the DoH resolver. Not only to not bother the upstream host,
  but maybe primarily because you don't want to wait longer for a
  response than necessary.
* DNSSEC validation. Even if your upstream DoH server claims to
  validate DNSSEC, can you trust it do so? Make unbound validate
  for you to be sure.
* Local domains. As unbound is the first point of contact for
  name resolution, you can make it prefer your own zones, either
  with local definitions or forward to a separate nameserver.

I don't know how much tor traffic is hitting the cloudflare
servers, but it may be possible to do traffic analysis and
somewhat deanonymize the queries. You can combine the tor
transport with a "round robin" rotation of public DoH providers
to further confuse the enemy.

So, in conclusion: encrypt everything, but realize that
encryption is not always everything that matters when it comes to
privacy. Check out [dohotcd][dohotcd] if my reasoning appeals to
you; see if it works for you or report issues.

(This post was originally posted in October 2019 in a more
verbose form. You can still read [that version here][oldpost].)

[rfc/8484]: https://tools.ietf.org/rfc/rfc8484.txt
[tor]: https://www.torproject.org/
[unbound]: https://nlnetlabs.nl/projects/unbound/about/
[cf/tor]: https://blog.cloudflare.com/welcome-hidden-resolver/
[py/doh-proxy]: https://facebookexperimental.github.io/doh-proxy/
[dohotcd]: https://github.com/olof/dohotcd
[oldpost]: https://github.com/olof/olof.github.io/blob/778eb010bda3e6ca4e6df285811d4deeb5322041/_posts/2019-10-22-dns-over-https-over-tor.md
