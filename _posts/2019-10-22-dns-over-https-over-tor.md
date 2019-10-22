---
layout: post
title: "DoH over Tor: Part 1: Background"
categories: dns, privacy, tor
---
This article series describes considerations for using
DNS-over-HTTPS (DoH, [RFC 8484][rfc/8484]) for confidentiality of
your DNS usage, and to further improve privacy properties by
using the [Tor anonymizing proxy network][tor].

## Background

DNS, the protocol that helps you resolve hostnames to IP
addresses has a privacy problem. All queries you do are sent on
the wire in the clear. Not only does your configured resolver see
your queries (that's expected!), but everything between you and
your resolver can also see the queries; for instance, your ISP.
Maybe you are using your ISP's DNS resolver; that's fine! In that
case, you are knowingly letting your ISP handle your DNS queries
and that may have some technical merits.  But even if you aren't,
maybe you are using your own resolver or a public resolver like
Cloudflare's 1.1.1.1 or Google's 8.8.8.8, your ISP may be
analyzing your query patterns.  Especially in the face of HTTPS
and encryption, your query patterns may be enough to figure out
what you are doing.

To verify integrity of a response (evidence that nobody has
manipulated the response), we have DNSSEC, where the DNS data is
*signed*. This has to be enabled for each domain by its owner and
even if the domain owner has enabled this, it doesn't solve the
issue of confidentiality: the queries are still sent in the open.
Personally, I have been doing local dns resolving using
[unbound][unbound], to enable trustless DNSSEC validation (i.e. I
don't want to trust a third party resolver to do this for me, I
want to keep it as close to the client as possible). This solves
problems related to data integrity, but again, does nothing to
solve confidentiality, as an ISP could simply inspect the traffic
en-route.

[DNS-over-HTTPS (DoH)][rfc/8484] is an IETF standardized solution
to avoid this. DoH is literally just what it sounds like: send a
binary DNS packet as the payload of an HTTPS request. The purpose
of this is to make use of the confidentiality provided by the
HTTPS encryption. It will also make it harder to identify it as
DNS traffic, and thereby block it (in theory it should just look
like regular HTTPS traffic). This protects against passive
eavesdropping, where somebody (like your ISP) can inspect your
DNS queries.

But DoH has faced some criticism. [Some of it][pdns/doh-blog]
points out some additional privacy leakage that can result from
using HTTP: e.g. leakage of User-Agent strings and cookies. This
is valid, but we can deal with it as long as we know about it.
Some of the complaints are at a conceptual level: about how it
gives *applications* (e.g. web browsers) more or less complete
control over name resolution, and means to bypass existing
network policy and monitoring. We shouldn't dismiss that
criticism, but some of those who most loudly voice this
conceptual complain are internet service providers (ISPs). And
the position they take is weird. If I'm not using my ISP's
resolver, they should not analyze my queries and thus remain
unaffected by changes like this. Yet, large ISPs have [made a
case to US congress][ncta/congress-letter] to investigate this
from an antitrust point of view: basically saying that "we can no
longer access data if it becomes encrypted". It's not their data
to access. There are valid reasons to monitor DNS queries: for
instance, it has been used to identify malware threats present in
a network. I don't think this usage is bad, but network operators
should not assume to have access to this information, and should
definitely not block efforts to improve DNS privacy just because
they may lose this access.

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
up a local DoH server that use it and point Firefox to that DoH
endpoint, and all would be well in a technical sense. But from a
network perspective, the flow of information has not changed: I
still delegate to my unbound to query using plain DNS. The
queries can thus still be correlated to my IP address. It does
not solve anything, privacy wise. Ok, maybe run your own DoH
server remotely then? This removes the ability for my local ISP
to track me, but the ISP of my remote DoH server can still see
the query patterns coming out of the resolver. If I'm the only
one using the resolver, it's the same as seeing *my* query
patterns. A little bit better maybe --- the queries can't be
correlated to my HTTP traffic, but still far from ideal. Instead,
maybe I *should* use one of the big DoH players' resolvers? That
way, I mix my queries with those of all other users. But I would
like to avoid letting the resolver operator know my query
patterns.

### DoH: confidentiality, Tor: anonymity

My idea to solve this was to proxy DoH requests over [Tor][tor], the
anonymizing proxy network. Turns out, Cloudflare hackers also [thought
about this][cf/tor]!

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

My first thought was to run a local reverse proxy; I'd configure
Firefox to use `http://localhost/dns-query` as my DoH endpoint
and it will then proxy the requests through Tor to a public DoH
resolver. This was a dead end: Firefox requires DoH hosts to be
HTTPS (and in fact, this requirement comes directly from [RFC
8484, Section 5][rfc/8484/5]). I can't get a valid certificate
for localhost. Maybe I could have gotten one for some other
public domain name, that pointed to 127.0.0.1, but not only is
this [not recommended][le/localhost], it means that everybody who
wants to set up something like this would need their own domain
name.

I also tried doing this with a [proxy auto
configuration][mdn/pac] (PAC) script, where I configure all
traffic aimed at e.g.
`https://mozilla.cloudflare-dns.com/dns-query` would go via the
proxy `socks5://127.0.0.1:9050`. But I couldn't make Firefox use
it, not sure why.

The only way I got Firefox to accept my custom DoH server was to
set it up on a publicly available host, with a domain name and a
proper certificate signed by Let's Encrypt. To enable this in
Firefox, you can use the Preferences UI or
[about:config][bagder/trrprefs] (the most important settings are
`network.trr.uri` and `network.trr.mode`). Doing so works, but it
has privacy implications: You are only moving the point you have
to trust from the DoH operator to the proxy operator who will see
all your queries and your IP address. No problem if you are the
only user. But it means that we can't share it with others and
still claim it has any significant privacy benefits.  Also,
creating a public attack surface to work around security design
is not ideal.

### Browsers only!?

Ok, so we can configure DoH for use in Firefox. Maybe some other
applications can also get modifications to support DoH. But why?
Name resolution has traditionally been a black box the operating
system provides for you. Changing this would come with huge costs
in software modifications, and sometimes it's not even possible.

For me, the proper place to actually implement DoH client support
is in the system resolver, or as close to it as possible. I run a
local unbound for name resolution, so using that as the
implementation point makes sense for me. I'm not an experienced C
developer, so adding support for DoH *in* unbound seems harder
than I could stomach right now (but it would probably be the
[Right Place(tm)][unbound/doh-bug]!). I also looked at PowerDNS
Recursor with its Lua scripting ability, but couldn't quite get
the behavior I wanted.

I also want to mention DNS-over-TLS (DoT, [RFC 7858][rfc/7858], a
more lightweight protocol modification. It uses a well known port
of 953. It makes sense and it's even already supported by
unbound, but the reason I went forward with DoH is that it also
solves the problem of "masking" the fact that it's a DNS query
(to some extent), making it harder to block. But this is a weak
argument, because it's likely that even encrypted, for the
sufficiently motivated network operator, DoH traffic is still
identifiable.  There [has been prior work][gh/piskyscan/dot-tor]
on tunneling DoT over Tor, please take a look if you're
interested.

### doh-proxy

Further research pointed me to the [doh-proxy python
project][py/doh-proxy] from Facebook. It includes software for running
your own DoH server, but it also includes `doh-stub` that acts like a
regular DNS resolver that forwards requests to a DoH server over
HTTPS. This is exactly the kind of building block I need; I can
configure unbound to forward queries to this stub resolver using
`forward-zone` statements.

The doh-proxy project is documented as experimental and it
unfortunately depends on an http2 python module that isn't well
maintained (currently no python3.7 support in a published
release) and it doesn't support fallback to http/1.1. doh-stub
doesn't support the use of proxies either, so were we to use it,
we would have to patch it. Even so, it allowed me to prototype
the communication chain I wanted:

```
Firefox -(dns)-> unbound -(dns)-> proxy -(   tor   )-> Doh resolver
                                          -(http)->
```

The proxy could be a patched doh-stub. I tried making it work
with the [Python requests library][py/requests] (it has support
for socks proxying), but that caused unacceptable performance
loss and instability as requests blocks the [asyncio event
loop][py/asyncio] waiting for a response. I could have patched it
to use the more mature [aiohttp][py/aiohttp] python library, but
instead, I implement my own doh-stub like tool.

## My implementation

I wrote a small stub resolver in Perl, using [AnyEvent][cpan/AnyEvent].

### Dependencies and configuration

Install packages:

```
apt-get install unbound tor privoxy libanyevent-perl \
                libanyevent-http-perl libanyevent-handler-udp-perl
```

#### Configure Privoxy

Configure Privoxy to forward traffic over tor, edit
/etc/privoxy/config and make sure you have the following line
uncommented:

```
forward-socks5t   /               127.0.0.1:9050    .
```

Normally, you can make Privoxy work for you in more ways, like
stripping privacy leakages, but since we'll only use it to proxy HTTPS
traffic Privoxy won't be able to very much as it's all encrypted. But
we do control the HTTP client, so we can take efforts to avoid leaking
privacy information that is normally associated with HTTP requests,
especially browser generated requests (no cookie jar, no HTTP
user-agent string).

Ironically, stripping away identifiable information from an HTTP
request can make the request more fingerprintable (e.g. not as many
requests lack a User-Agent string and come from a Tor endpoint). I
hope I can make the DoH over Tor proxy work well enough to be widely
usable *or* through these blog posts inspire somebody else to do so.

#### Configure unbound

Our resolver stub will run on the non-standard port 5354/udp.
This makes it possible to run it without special privileges, and
it also makes it possible to run another resolver on port 53/udp.
This other resolver, in my case, is unbound. I can make it
forward all queries to the stub resolver using `forward-zone`
configuration blocks. There are some important reasons to do so:

* Caching! You *don't* want to forward every query over and over
  to the DoH resolver. Not only to not bother the upstream host,
  but maybe primarily because you don't want to wait longer for a
  response than necessary.
* DNSSEC validation. Even if your upstream DoH server validates
  DNSSEC, can you trust it do so? Validate on your host, using a
  validating resolver like unbound.
* Local domains. As unbound is the first point of contact for
  name resolution, you can make it prefer your own zones, either
  with local definitions or forward to a separate nameserver.

The minimal config to make use of the doh proxy resolver in
unbound:

```
server:
    do-not-query-localhost: no

forward-zone:
    name: .
    forward-addr: ::1@5354
```

`forward-zone` configures unbound to forward all our queries to
the recursive resolver specified in `forward-addr`. We do it for
all names (anything under `.`).

`do-not-query-localhost` bypasses some security(?) thing in
unbound since we run our forwarder on localhost. Not sure exactly
what the implications are, but it's necessary for unbound to pass
on queries.

Because resolving names over Tor is slower (sometimes much
slower), we can also tweak unbound to respond with expired names
(i.e where we got data in the cache, but the TTL has passed),
while still updating the cache once the DoH response has been
received.  This may have some terrible consequence that I haven't
thought about. I'm still trying it out :).

```
server:
    serve-expired: yes
```

### Code

Here follows a simplified version for demonstrating the concept.
I aim to productify it a bit and publish it properly.

* Listen on [::1]:5354/udp for incoming DNS queries
* For each incoming query, forward it to using an HTTP POST to
  the configured DoH endpoint. Do so over Tor.
* When a response is received just send it back to the DNS
  client.

```perl
use strict;
use warnings;
use AnyEvent;
use AnyEvent::Handle::UDP;
use AnyEvent::HTTP;

my $uri = 'https://1.1.1.1/dns-query';
my $proxy = 'http://127.0.0.1:8118';

AnyEvent::HTTP::set_proxy($proxy);

my $named = AnyEvent::Handle::UDP->new(
    bind => ['::1', 5354],
    on_recv => sub {
        # Called when a new UDP packet has been received, forward
        # dns query to the configured HTTP endpoint, $uri.
        my ($data, $ae, $client) = @_;
        http_post $uri, $data,
            headers => {
                Accept => 'application/dns-message',
                'User-Agent' => '',
                'Content-Type' => 'application/dns-message',
                'Content-Length' => length($data),
            },
            tls_ctx => 'high',
            persistent => 1,
            sub {
                # Called when HTTP query completes, send back response
                # payload to our client
                my ($data, $headers) = @_;
                $ae->push_send(shift, $client)
            };
    },
);

my $w = AnyEvent->condvar;
$w->recv;
```

Some notes about AnyEvent::HTTP: By default, it won't properly
validate TLS certificates. Passing the `tls_ctx => 'high'`
parameter enables this. To have to opt-in for this is not really
ideal for an HTTPS client, but now we know about it and we have
to make sure to always enable it.

Note also that we didn't use a domain name to specify the
upstream DoH resolver. Because of a chicken and egg problem, we
can't use a domain name here: just like you can't use a domain
name in /etc/resolv.conf. To resolve the domain name of your
resolver you would have to resolve the domain name of your
resolver. Firefox solved this by having a "bootstrap address",
where you can define a name but still use a hardcoded IP address.

We also take advantage of persistent connections; this means that
once a query is made, the connection stays active for some small
amount of time, in case we want to send more queries in quick
succession.

#### Enhanced logging with Net::DNS

For logging or statistics, it may be useful to include
information from the actual packet. You can parse the incoming
(or outgoing) packets with Net::DNS:

```perl
use Net::DNS::Packet;
...

my $pkt = Net::DNS::Packet->new(\$data);

my $id = $pkt->header->id;
my @names = map { $_->name } $pkt->question;
```

Because this is running on localhost, you can add extensive
logging without this becoming an immediate privacy issue
(assuming deployment on non-shared hosts). I would like to do
that as a means of analyzing my own query patterns, but I don't
want to conflate it with the rest of the article.  Maybe in some
later, upcoming post, yes?

### Future work

I use HTTP/1.1 transport, but [RFC 8484][rfc/8484/5.2] specifies
HTTP/2 as the "minimum RECOMMENDED version". I would like to see
the kind of performance improvements we could get by adding
support for HTTP/2. I would also like to see if we can optimize
the HTTP persistency timeouts. I also want to run benchmarks; how
much slower is this as compared to running a) regular DNS
resolution, b) regular DoH usage.

On top of this plan to productify the code, add system
integration like initscripts/systemd service files. I will move
hardcoded settings in the code to configuration files.

## Conclusions

While some things are still missing, this article presents a way
to make use of DoH with Tor. By implementing this as part of the
system resolver, it does so in a way independent of specific
applications (e.g. no need to configure Firefox to use it). I
hope to return with followup posts with results from benchmarks
and announcement of a first release.

[rfc/7830]: https://tools.ietf.org/html/rfc7830
[rfc/7858]: https://tools.ietf.org/html/rfc7858
[rfc/8484]: https://tools.ietf.org/rfc/rfc8484.txt
[rfc/8484/5]: https://tools.ietf.org/html/rfc8484#section-5
[rfc/8484/5.2]: https://tools.ietf.org/html/rfc8484#section-5.2
[tor]: https://www.torproject.org/
[pdns/doh-blog]: https://blog.powerdns.com/2019/09/25/centralised-doh-is-bad-for-privacy-in-2019-and-beyond/
[unbound]: https://nlnetlabs.nl/projects/unbound/about/
[unbound/doh-bug]: https://web.archive.org/web/20190625135131/https://www.nlnetlabs.nl/bugs-script/show_bug.cgi?id=1200
[gh/curl/dohlist]: https://github.com/curl/curl/wiki/DNS-over-HTTPS#publicly-available-servers
[gh/piskyscan/dot-tor]: https://github.com/piskyscan/dns_over_tls_over_tor
[cf/tor]: https://blog.cloudflare.com/welcome-hidden-resolver/
[ncta/congress-letter]: https://www.ncta.com/sites/default/files/2019-09/Final%20DOH%20LETTER%209-19-19.pdf
[mdn/pac]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Proxy_servers_and_tunneling/Proxy_Auto-Configuration_(PAC)_file
[bagder/trrprefs]: https://bagder.github.io/TRRprefs/
[py/requests]: https://3.python-requests.org/
[py/asyncio]: https://docs.python.org/3/library/asyncio.html
[py/doh-proxy]: https://facebookexperimental.github.io/doh-proxy/
[py/aiohttp]: https://aiohttp.readthedocs.io/en/stable/
[le/localhost]: https://letsencrypt.org/docs/certificates-for-localhost/
[cpan/AnyEvent]: https://metacpan.org/pod/AnyEvent
[cpan/AnyEvent::Handle::UDP]: https://metacpan.org/pod/AnyEvent::Handle::UDP
[cpan/AnyEvent::HTTP]: https://metacpan.org/pod/AnyEvent::HTTP
