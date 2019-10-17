---
layout: post
title: "DoH over Tor: Part 3: Implementation"
categories: dns, privacy
---
This article will discuss a simple implementation of DoH+Tor stub
resolver in Perl. It will listen for incoming DNS requests and
forward them, via Tor, to a public DoH resolver. This way, we get
the privacy properties of HTTPS for all DNS resolution on the
system.

In the [last post][self/doh-part-2], I described the chain I
wanted:

```
Firefox -(dns)-> unbound -(dns)-> proxy -(doh proxied over tor)-> Doh resolver
```

I will focus on the proxy step in this article. What would this
look like? This is not a detailed tutorial; if you are unable to
follow along, perhaps you should just wait until something like
this is nicely packaged for your distribution. With that said,
let me know if you have problems with this setup. I have tested
this on Ubuntu 18.10.

### Basic setup

Install packages:

```
apt-get install tor privoxy libnet-dns-perl libanyevent-perl \
                libanyevent-http-perl libanyevent-handler-udp-perl
```

Configure privoxy to forward traffic over tor, edit
/etc/privoxy/config and make sure you have the following line
uncommented:

```
forward-socks5t   /               127.0.0.1:9050    .
```

Normally, you can make privoxy work for you in more ways, like
stripping privacy leakages, but since we'll only use it to proxy
HTTPS traffic privoxy won't be able to very much as it's all
encrypted. But we do control the HTTP client, k

### Implementation

A first go at it: Using [AnyEvent][cpan/AnyEvent] and
[AnyEvent::Handle::UDP][cpan/AnyEvent::Handle::UDP], we set up a
resolver that accepts UDP datagrams. On each incoming packet, we
forward the payload (a regular DNS packet) to the HTTP endpoint
listed in `$uri`, via the proxy specified in `$proxy`.  The
payload of the HTTP response (again, a normal DNS packet) is sent
back to the original DNS querier.

We use [AnyEvent::HTTP][cpan/AnyEvent::HTTP] as HTTP client, but
since it doesn't support socks proxies, we use the http proxy
provided by privoxy.  (They do document a way to hack in socks
support, but we'll go with privoxy for now.)

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

This is the first simplified attempt, without any efforts to make
it "production ready", but the implementation is really simple
and illustrates the overall handling. Making it more production
ready would include reading a config file (not hardcoding
Cloudflare's IP address!), logging and maybe automated tests. My
plan is to clean it up and publish somewhere.

For logging, it may be useful to include information from the
actual packet. You can parse the incoming (or outgoing) packets
with Net::DNS:

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

Note also that we didn't use a domain name to specify the
upstream DoH resolver. Because there's a chicken and egg problem
here, we can't use a domain name here: just like you can't use a
domain name in /etc/resolv.conf. You would have to resolve the
name first, and to contact the resolver you would have to resolve
the name. Firefox solved this by having a "bootstrap address",
where you can define a name but still use a hardcoded IP address.

#### Tweaking AnyEvent::HTTP

Some notes about AnyEvent::HTTP: By default, it won't validate
tls certificates. Passing the `tls_ctx => 'high'` parameter
enables this. To have to opt-in for this is not really ideal for
an HTTPS client, but now we know about it and we have to make
sure to always enable it.

AnyEvent::HTTP doesn't do persistent connections by default
either. Not as big deal to have to opt-in for: just add
`persistent => 1`. There are some tweakables related to this,
e.g. `$AnyEvent::HTTP::PERSISTENT_TIMEOUT`. I haven't optimized
this, I just set to some arbitrary values for now.

### Unbound configuration

You may have noticed that the above script listens on 5354/udp,
not the regular dns port 53/udp. This is correct; it makes it
easier to run as an unprivileged user. But it also makes it
easier to run a "real" resolver on port 53. I run unbound on my
host, and I want to make it forward incoming queries to my doh
proxy resolver.

There are some important reasons to do so:

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
slower), we can tweak unbound to respond with expired names (i.e
where we got data in the cache, but the TTL has passed), while
still updating the cache once the DoH response has been received.
This may have some terrible consequence that I haven't thought
about. I'm still trying it out :).

```
server:
    serve-expired: yes
```

### Conclusion

This post implements a simple doh tor proxy. I'm currently
running it locally, and name resolution does feel slower. It does
not feel unacceptably slow though, given the privacy benefits it
provides. I'm still looking into some instability issues,
especially when faced with network failures; Unbound will in some
cases cache servfails caused by the DoH proxy not being able to
connect to its upstream. (Perhaps it even blacklist it if it
behaves badly?) I have some more work to do before I can publish
it as its own software project. But that's my aim. For now, I'll
let it rest for a bit while using it to smoke out instability
issues.

The next article in this series will probably be published in
some weeks. Until then, I'll try to make some performance
comparisons (how much does Tor or HTTP client tweakables affect
performance?), tidy up the software packaging and make sure to
tweak necessary unbound configuration. Stay tuned.

And if you can't wait, try it out! Copy the code and run it; but
on the other hand, it's some random code on some random blog.
You can't really assume it's maintained. But please contact me
anyways somehow if you find problems!

Other posts in this series:

* [Part 1: Background][self/doh-part-1]
* [Part 2: Considerations][self/doh-part-2]

[self/doh-part-1]: https://blog.3.14159.se/posts/2019/10/15/dns-over-https-over-tor-part1
[self/doh-part-2]: https://blog.3.14159.se/posts/2019/10/15/dns-over-https-over-tor-part2
[cpan/AnyEvent]: https://metacpan.org/pod/AnyEvent
[cpan/AnyEvent::Handle::UDP]: https://metacpan.org/pod/AnyEvent::Handle::UDP
[cpan/AnyEvent::HTTP]: https://metacpan.org/pod/AnyEvent::HTTP
