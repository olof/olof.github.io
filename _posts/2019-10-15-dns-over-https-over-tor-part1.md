---
layout: post
title: "DoH over Tor: Part 1: Background"
categories: dns, privacy, tor
---
This article series describes considerations for using
DNS-over-HTTPS (DoH, [RFC 8484][rfc/8484]) for confidentiality of
your DNS usage, and to further improve privacy properties by
using the [Tor anonymizing proxy network][tor].

### DNS confidentiality and DNS-over-HTTPS

DoH is literally just what it sounds like: send a binary DNS
packet as the payload of an HTTPS request. The purpose of this is
to make use of the confidentiality provided by the HTTPS
encryption. It will also make it harder to identify it as DNS
traffic, and thereby block it (in theory it should just look like
regular HTTPS traffic). This protects against passive
eavesdropping, where somebody (like your ISP) can inspect your
DNS queries. My idea is to further increase the privacy
properties of this by hiding the source IP address (i.e.
requestor identity) from the resolver operator.

The root problem is that when you do DNS resolution, the queries
are sent in the open; they can be intercepted by your ISP, but
even without intercepting them, they can still be monitored and
analyzed. Maybe you are using your ISP's DNS resolver; that's
fine! In that case, you are knowingly letting your ISP handle
your DNS and that may have some technical merits. But even if you
aren't, like when you are using your own resolver or a public
resolver like Cloudflare's 1.1.1.1 or Google's 8.8.8.8, your ISP
may be analyzing your query patterns. Especially in the face of
HTTPS and encryption, your query patterns may be enough to figure
out what you are doing.

To verify integrity of a response, we have DNSSEC, where the DNS
data is *signed*. This has to be enabled for each domain and even
if the domain owner has enabled this, it doesn't solve the issue
of confidentiality. That's where DoH comes in (and other
suggested solutions like DoT or dnscrypt). But DoH has faced some
criticism. [Some of it][pdns/doh-blog] points out some additional
privacy leakage that can result from using HTTP: e.g. leakage of
User-Agent strings and cookies. This is valid, but we can deal
with it as long as we know about it. Some of the complaints are
at a conceptual level: about how it gives browsers more or less
complete control over name resolution, and means to bypass
existing network policy and monitoring.

We shouldn't dismiss that criticism, but some of those that most
loudly voice this conceptual complain are internet service
providers (ISPs). And here's where it becomes a bit weird. If I'm
not using my ISP's resolver, they should not analyze my queries
and thus remain unaffected by changes like this. Yet, large ISPs
have [made a case to US congress][ncta/congress-letter] to
investigate this from an antitrust point of view: "we can no
longer access data if it becomes encrypted". They most definitely
should not care.

Personally, I have been doing local dns resolving using
[unbound][unbound], to enable trustless DNSSEC validation (i.e. I
don't want to trust a third party resolver to do this for me, I
want to keep it as close to the client as possible). This solves
problems related to data integrity, but again, does nothing to
solve confidentiality, as an ISP could simply inspect the traffic
en-route.

I don't want to guess why Google is pushing for DoH, maybe it's
all in good faith --- an effort to improve privacy on the
Internet. Or maybe it's because they get access to even more DNS
queries (and thus user behavioral data). That's besides the
point. The protocol itself is not harmful. The important thing is
that you have control over which DoH endpoint to use. Use
Cloudflare, use Google, Quad9 or use [any DoH compatible
resolver][gh/curl/dohlist]. The choice should be yours. Having a
choice should also mean thinking about the consequences.

I could muck around with my own setup with a local unbound, set
up a local DoH server that use it and point firefox to that DoH
endpoint, and all would be well in a technical sense. But from a
network perspective, the flow of information has not changed: I
still delegate to my unbound to query using plain DNS. The
queries can thus still be correlated to my IP address. It does
not solve anything, privacy wise.

Ok, maybe run your own DoH server remotely? This removes the
ability for my local ISP to track me, but the ISP of my remote
DoH server can still see the query patterns coming out of the
resolver. If I'm the only one using the resolver, it's the same
as seeing *my* query patterns. A little bit better maybe --- the
queries can't be correlated to my HTTP traffic, but still far
from ideal. Instead, maybe I *should* use one of the big DoH
players' resolvers? That way, I mix my queries with those of all
other users. But I would like to avoid letting the resolver
operator know my query patterns.

### DoH: confidentiality, Tor: anonymity

My idea to solve this was to proxy DoH requests over Tor. Turns
out, Cloudflare hackers also [thought about this][cf/tor]!

> the exceptionally privacy-conscious folks might not want to
> reveal their IP address to the resolver at all, and we respect
> that. This is why we are launching a Tor onion service for our
> resolver at
> dns4torpnlfs2ifuz2s2yf3fc7rdmsbhm6rw75euj35pac6ap25zgqad.onion
> and accessible via tor.cloudflare-dns.com.

("the exceptionally privacy-conscious"? Let's make this the
norm!) But sadly this Cloudflare Onion service is no longer
online. Had this been online, things would have been a bit
simpler. But I still want this anonymity from the DNS provider,
so what I will do is somehow proxy the DoH traffic over Tor
myself.

In the next part of this series, I'll discuss methods of doing so
and reason about their relative advantages and disadvantages.

* [Part 2: Considerations][self/doh-part-2]
* [Part 3: Implementation][self/doh-part-3]

[rfc/8484]: https://tools.ietf.org/rfc/rfc8484.txt
[tor]: https://www.torproject.org/
[pdns/doh-blog]: https://blog.powerdns.com/2019/09/25/centralised-doh-is-bad-for-privacy-in-2019-and-beyond/
[unbound]: https://nlnetlabs.nl/projects/unbound/about/
[gh/curl/dohlist]: https://github.com/curl/curl/wiki/DNS-over-HTTPS#publicly-available-servers
[cf/tor]: https://blog.cloudflare.com/welcome-hidden-resolver/
[self/doh-part-2]: https://blog.3.14159.se/posts/2019/10/15/dns-over-https-over-tor-part2
[self/doh-part-3]: https://blog.3.14159.se/posts/2019/10/16/dns-over-https-over-tor-part3
[ncta/congress-letter]: https://www.ncta.com/sites/default/files/2019-09/Final%20DOH%20LETTER%209-19-19.pdf
