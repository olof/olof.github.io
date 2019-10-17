---
layout: post
title: "DoH over Tor: Part 2: Considerations"
categories: dns, privacy
---
The privacy properties of DNS-over-HTTPS are nice. I want to make
use of them! As I discussed in the [DoH
background][self/doh-part-1] article, Cloudflare did provide
their DoH over Tor, but not anymore.

**THIS IS A DRAFT, REVIEW COMMENTS WELCOME**

The DoH hype is focused on the web browsers; both Mozilla and
Google have come a long way in enabling it by default for Firefox
and Chrome. I've looked at how Firefox deals with DoH, and what
requirements it puts on the resolver.

My first thought was to run a local reverse proxy; I'd configure
Firefox to use http://localhost/dns-query as my DoH endpoint and
it will then proxy the requests through Tor to a public DoH
resolver. This was a dead end: Firefox requires DoH hosts to be
HTTPS (and in fact, this reuqirement comes directly from [RFC
8484, Section 5][rfc/8484/5]).

I can't get a valid certificate for localhost. Maybe I could have
gotten one for some other public domain name, that pointed to
127.0.0.1, but not only is this [not recommended][le/localhost],
it means that everybody who wants to set up something like this
would need their own domain name. I also considered doing this
with a [proxy auto configuration][mdn/pac] (PAC) script, but I
couldn't make Firefox use it.

The only way I got Firefox to accept my custom DoH server was to
set it up on a publicly available host, with a domain name and a
proper certificate signed by Let's Encrypt. Doing so could work,
but it has privacy implications: You are only moving the point
you have to trust from the DoH operator to the proxy operator.
The proxy operator sees your queries and your IP address. No
problem if you are the only user. But it means that we can't
share it with others and still claim it has any significant
privacy benefits. Also, creating a public attack surface to work
around security design... meh.

To enable this in Firefox, you can use the Preferences UI or
[about:config][bagder/trrprefs] (the most important settings are
`network.trr.uri` and `network.trr.mode`).

## But all the other applications?

Ok, so we can configure DoH for use in Firefox. Maybe some other
applications can also get support for DoH. But why? Name
resolution has traditionally been a black box the operating
system provides for you. Changing this would come with huge costs
in changing software, and sometimes it's not even possible.

For me, the proper place to actually implement DoH client support
is in the system resolver, or as close to it as possible. I run a
local unbound for name resolution, so using that as the
implementation point makes sense for me.

### DNS-over-TLS

I also want to mention DNS-over-TLS (DoT, [RFC 7858][rfc/7858], a
more lightweight protocol modification. It makes sense, but the
reason I went forward with DoH is that it also solves the problem
of "masking" the fact that it's a DNS query (to some extent). But
this is a weak argument, because it's likely that DoH traffic is
also identifiable even if encrypted, at least if you use one of
the popular public DoH resolvers.

There [has been prior work][gh/piskyscan/dot-tor] on tunneling
DoT over Tor, please take a look if you're interested.

### Implementation, a first look

I'm not an experienced C developer, so adding support for DoH
*in* unbound seems harder than I could stomach right now (but it
would probably be the [Right Place(tm)][unbound/doh-bug]!). I
also looked at PowerDNS Recursor with its Lua scripting ability,
but couldn't quite get the behavior I wanted. Further research
pointed me to the [doh-proxy python project][py/doh-proxy] from
Facebook. It includes software for running your DoH server, but
it also includes `doh-stub` that acts like a regular DNS resolver
that forwards requests to a DoH server over HTTPS.

This is exactly the kind of building block I need; I can
configure unbound to forward queries to this stub resolver using
`forward-zone` statements. The doh-proxy project is documented as
experimental and it unfortunately depends on an http2 python
module that isn't well maintained (no python3.7 support in a
published release) and it doesn't support fallback to http/1.1.
It doesn't support the use of proxies either, so were we to use
it, we would have to patch it.

So, given this, I think the setup I would like to see is:

```
Firefox -(dns)-> unbound -(dns)-> proxy -(doh proxied over tor)-> Doh resolver
```

The proxy could be a patched doh-stub. I tried making it work
with the [Python requests library][py/requests] (it has support
for socks proxying), but that caused unacceptable performance
loss and instability as requests blocks the [asyncio event
loop][py/asyncio] waiting for a response. I could have patched it
to use the more mature [aiohttp][py/aiohttp] python library, but
instead, I implement my own doh-stub like tool. I'll go into
details on this in the next couple of posts.

Other posts in this series:

* [Part 1: Background][self/doh-part-1]
* [Part 3: Implementation][self/doh-part-3]

[mdn/pac]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Proxy_servers_and_tunneling/Proxy_Auto-Configuration_(PAC)_file
[self/doh-part-1]: https://blog.3.14159.se/posts/2019/10/15/dns-over-https-over-tor-part1
[self/doh-part-3]: https://blog.3.14159.se/posts/2019/10/16/dns-over-https-over-tor-part3
[rfc/8484/5]: https://tools.ietf.org/html/rfc8484#section-5
[rfc/7858]: https://tools.ietf.org/html/rfc7858
[bagder/trrprefs]: https://bagder.github.io/TRRprefs/
[unbound/doh-bug]: https://web.archive.org/web/20190625135131/https://www.nlnetlabs.nl/bugs-script/show_bug.cgi?id=1200
[py/requests]: https://3.python-requests.org/
[py/asyncio]: https://docs.python.org/3/library/asyncio.html
[py/doh-proxy]: https://facebookexperimental.github.io/doh-proxy/
[py/aiohttp]: https://aiohttp.readthedocs.io/en/stable/
[gh/piskyscan/dot-tor]: https://github.com/piskyscan/dns_over_tls_over_tor
[le/localhost]: https://letsencrypt.org/docs/certificates-for-localhost/
