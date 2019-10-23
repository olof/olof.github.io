---
layout: post
title: Continuous monitoring of DNS with tcpdump
categories: dns, privacy, monitoring, packetq
---
This post describes one idea for continuously monitoring local
DNS behavior based on tcpdump. It also discusses the
[PacketQ][packetq] tool for analyzing the generated pcap capture
files.

I'm evaluating this in parallell to another solution I'll surely
post about in the future based on Statsd + Grafana; if nothing
else, I'll be able to crossvalidate the data to identify
unexpected differences.

My work in this area is motivated by a wish to understand *what I
leak*. If you want to spy on your users/customers/citizens, the
work should be adaptable to that use case as well. Good luck and
have fun with that. There's probably better alternatives for that
though.

## Capturing DNS packets

I wrote a super simple service for doing continuous network
captures of DNS traffic using tcpdump, `/usr/sbin/dns-monitor`.
It takes the interface to monitor as argument.

```shell
#!/bin/sh
if=$1
mkdir -p /var/lib/dns-monitor
tcpdump -G 86400 -w /var/lib/dns-monitor/"$if".%Y%m%d_%H%M.pcap -i"$if" port 53
```

Make sure it has exec permission. This is combined with a systemd
service template, `dns-monitor@.service`:

```
[Unit]
Description=dns monitor
Wants=network.target

[Service]
ExecStart=/usr/sbin/dns-monitor %i

[Install]
WantedBy=multi-user.target
```

[Systemd service templates][systemd/unit] are instantiated by
appending a parameter after the @ sign. In this case, it's
parameterized with the interface name. You can enable this for
localhost by doing:

```
$ systemctl enable dns-monitor@lo.service
$ systemctl start dns-monitor@lo.service
```

It does run as root (because it needs to be able to sniff network
traffic), but could be handled using capabilites instead. I leave
it as an excercise for the reader.

This will create pcap capture files in `/var/lib/dns-monitor`
like `lo.20191023_1127.pcap`. We also want to make sure the
captures are cleaned up after a while (both because of storage
*and* privacy considerations; retention period is up to you, I
use 30 days). Create a cronjob `/etc/cron.daily/dns-monitor`
(make sure it has exec permission):

```shell
#!/bin/sh
find /var/lib/dns-monitor/*.pcap -type f -mtime +30 -delete
```

## Analyzing captures with PacketQ

So, we have a bunch of pcap files, and we can have fun with them
in wireshark or whatever. But I also want to mention the PacketQ
tool, providing an SQL like interface for pcap files of
(primarily) dns packets. It was initially developed by
[Internetstiftelsen][iis], the Swedish TLD operator, but has
since been adopted by [DNS-OARC][oarc/packetq].

```
$ packetq -s 'select * from dns' /var/lib/dns-monitor/lo.20191023_1127.pcap
[
  {
    "table_name": "result-0",
    "query": "select * from dns",
    "head": [
      { "name": "id","type": "int" },
      { "name": "s","type": "int" },
      { "name": "us","type": "int" },
      { "name": "ether_type","type": "int" },
      { "name": "src_port","type": "int" },
      { "name": "dst_port","type": "int" },
      { "name": "src_addr","type": "text" },
      { "name": "dst_addr","type": "text" },
      { "name": "protocol","type": "int" },
      { "name": "ip_ttl","type": "int" },
      { "name": "ip_version","type": "int" },
      { "name": "fragments","type": "int" },
      { "name": "qname","type": "text" },
      { "name": "aname","type": "text" },
      { "name": "msg_id","type": "int" },
      { "name": "msg_size","type": "int" },
      { "name": "opcode","type": "int" },
      { "name": "rcode","type": "int" },
      { "name": "extended_rcode","type": "int" },
      { "name": "edns_version","type": "int" },
      { "name": "z","type": "int" },
      { "name": "udp_size","type": "int" },
      { "name": "qd_count","type": "int" },
      { "name": "an_count","type": "int" },
      { "name": "ns_count","type": "int" },
      { "name": "ar_count","type": "int" },
      { "name": "qtype","type": "int" },
      { "name": "qclass","type": "int" },
      { "name": "atype","type": "int" },
      { "name": "aclass","type": "int" },
      { "name": "attl","type": "int" },
      { "name": "aa","type": "bool" },
      { "name": "tc","type": "bool" },
      { "name": "rd","type": "bool" },
      { "name": "cd","type": "bool" },
      { "name": "ra","type": "bool" },
      { "name": "ad","type": "bool" },
      { "name": "do","type": "bool" },
      { "name": "edns0","type": "bool" },
      { "name": "qr","type": "bool" },
      { "name": "edns0_ecs","type": "bool" },
      { "name": "edns0_ecs_family","type": "int" },
      { "name": "edns0_ecs_source","type": "int" },
      { "name": "edns0_ecs_scope","type": "int" },
      { "name": "edns0_ecs_address","type": "text" }
    ],
    "data": [
      [1,1571823089,546025,2048,33253,53,"127.0.0.1","127.0.0.1",17,64,4,0,"connectivity-check.ubuntu.com.","",16589,47,0,0,0,0,0,0,1,0,0,0,1,1,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,""],
      [2,1571823089,546430,2048,53,33253,"127.0.0.1","127.0.0.1",17,64,4,0,"connectivity-check.ubuntu.com.","connectivity-check.ubuntu.com.",16589,79,0,0,0,0,0,0,1,2,0,0,1,1,1,1,50,0,0,1,0,1,0,0,0,1,0,0,0,0,""],
      ...
    ]
  }
]
```

It can also export to CSV or XML, as well as expose a simple web
interface via its builtin web server. We can use the capture
files and packetq to generate reports like most queried domains.

By having these pcap files around, we can later write tools to
create reports. For instance, the following expression will
generate a list of most popular domains by number of replies
(based on an expression listed in the [PacketQ
FAQ][packetq/faq]).

```
$ packetq -s "select aname, count(*) as count from dns where qr=1 group by aname order by count desc limit 10"
[
  {
    "table_name": "result-0",
    "query": "select aname, count(*) as count from dns where qr=1 and rcode=0 group by aname order by count desc limit 10",
    "head": [
      { "name": "aname","type": "text" },
      { "name": "count","type": "int" }
    ],
    "data": [
      ["",301],
      ["connectivity-check.ubuntu.com.",67],
      ["safebrowsing.googleapis.com.",22],
      ["live.github.com.",16],
      ["ac.duckduckgo.com.",11],
      ["www.dns-oarc.net.",10],
      ["www.google.com.",10],
      ["detectportal.firefox.com.",10],
      ["aws.amazon.com.",10],
      ["support.mozilla.org.",8]
    ]
  }
]
```

You can do much more with PacketQ, but I'll end my post here. I'm
yet to do something genuinely useful with it, but having the
tcpdump service in place and active will help me collect a
dataset for later analysis.

[packetq]: https://github.com/DNS-OARC/PacketQ
[oarc/packetq]: https://www.dns-oarc.net/tools/packetq
[packetq/faq]: https://github.com/DNS-OARC/PacketQ/blob/master/FAQ.md
[systemd/unit]: https://www.freedesktop.org/software/systemd/man/systemd.unit.html
[iis]: https://internetstiftelsen.se/
