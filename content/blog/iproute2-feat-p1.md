---
Title: Discovering iproute2 features (Part I)
Date: 2018-05-28
---

`iproute2` is the standard networking toolbox for Linux, deprecating the
unmaintained `net-tools`. `iproute2` includes the known `ip` tool, which most
admins and engineers use almost every day in order to see and manipulate a
host's network configuration. Today, I'm going to present some nice (and
sometimes undocumented) features of `ip` that I was not aware of until lately.

The following commands were all tested on a Debian Stretch box, running
iproute2 version `4.14.1-1~bpo9+1` (from stretch-backports) or
`iproute2-ss171113`.

You're all familiar with iproute2 commands like these, right?

```
# ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eno1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN mode DEFAULT group default qlen 1000
    link/ether 11:22:33:44:55:66 brd ff:ff:ff:ff:ff:ff
3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
    link/ether 00:aa:bb:cc:dd:cc brd ff:ff:ff:ff:ff:ff

# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
    link/ether 11:22:33:44:55:66 brd ff:ff:ff:ff:ff:ff
3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:aa:bb:cc:dd:cc brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.6/24 brd 192.168.2.255 scope global dynamic wlp3s0
       valid_lft 84101sec preferred_lft 84101sec
    inet6 fe80::1234:5678:9abc:def0/64 scope link
       valid_lft forever preferred_lft forever
```

Output looks a little bit ugly, right? What if you only want to know
info about your IPs, quickly? Let's try the `-br` and `-c` flags:

```
# ip -br -c a
lo               UNKNOWN        127.0.0.1/8 ::1/128
eno1             DOWN
wlp3s0           UP             192.168.2.6/24 fe80::1234:5678:9abc:def0/64

# ip -br -c l
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
eno1             DOWN           11:22:33:44:55:66 <NO-CARRIER,BROADCAST,MULTICAST,UP>
wlp3s0           UP             00:aa:bb:cc:dd:cc <BROADCAST,MULTICAST,UP,LOWER_UP>
```

* Pros: Way more readable and flashy :)
* Cons: Works only with the above commands

Quite better, right? But what if you need to parse something quickly without
using C or those ugly netlink wrappers in Python?

```
# ip -json a | jq .
[
  {
    "ifindex": 1,
    "ifname": "lo",
    "flags": [
      "LOOPBACK",
      "UP",
      "LOWER_UP"
    ],
    "mtu": 65536,
    "qdisc": "noqueue",
    "operstate": "UNKNOWN",
    "group": "default",
    "txqlen": 1000,
    "link_type": "loopback",
    "address": "00:00:00:00:00:00",
    "broadcast": "00:00:00:00:00:00",
    "addr_info": [
      {
        "family": "inet",
        "local": "127.0.0.1",
        "prefixlen": 8,
        "scope": "host",
        "label": "lo",
        "valid_life_time": 4294967295,
        "preferred_life_time": 4294967295
      },
      {
        "family": "inet6",
        "local": "::1",
        "prefixlen": 128,
        "scope": "host",
        "valid_life_time": 4294967295,
        "preferred_life_time": 4294967295
      }
    ]
  },
  {
    "ifindex": 2,
    "ifname": "eno1",
    "flags": [
      "NO-CARRIER",
      "BROADCAST",
      "MULTICAST",
      "UP"
    ],
    "mtu": 1500,
    "qdisc": "pfifo_fast",
    "operstate": "DOWN",
    "group": "default",
    "txqlen": 1000,
    "link_type": "ether",
    "address": "11:22:33:44:55:66",
    "broadcast": "ff:ff:ff:ff:ff:ff",
    "addr_info": []
  },
  {
    "ifindex": 3,
    "ifname": "wlp3s0",
    "flags": [
      "BROADCAST",
      "MULTICAST",
      "UP",
      "LOWER_UP"
    ],
    "mtu": 1500,
    "qdisc": "mq",
    "operstate": "UP",
    "group": "default",
    "txqlen": 1000,
    "link_type": "ether",
    "address": "00:aa:bb:cc:dd:cc",
    "broadcast": "ff:ff:ff:ff:ff:ff",
    "addr_info": [
      {
        "family": "inet",
        "local": "192.168.2.6",
        "prefixlen": 24,
        "broadcast": "192.168.2.255",
        "scope": "global",
        "dynamic": true,
        "label": "wlp3s0",
        "valid_life_time": 83904,
        "preferred_life_time": 83904
      },
      {
        "family": "inet6",
        "local": "fe80::1234:5678:9abc:def0",
        "prefixlen": 64,
        "scope": "link",
        "valid_life_time": 4294967295,
        "preferred_life_time": 4294967295
      }
    ]
  }
]
```

* Pros: Finally, something parsable
* Cons: Works only for `ip {,-s} {a,l} show`. Not documented. 

Also, you like to use iproute2 to check for metrics about your
interfaces, right? So, you must be familiar with this command:

```
# ip -s l show wlp3s0
3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
    link/ether 00:aa:bb:cc:dd:cc brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast
    199912659  156191   0       0       0       0
    TX: bytes  packets  errors  dropped carrier collsns
    14584622   102498   0       0       0       0
```

Let's add one `-s` more:

```
# ip -s -s l show wlp3s0
3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
    link/ether 00:aa:bb:cc:dd:cc brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast
    202179501  157989   0       0       0       0
    RX errors: length   crc     frame   fifo    missed
               0        0       0       0       0
    TX: bytes  packets  errors  dropped carrier collsns
    14754983   103641   0       0       0       0
    TX errors: aborted  fifo   window heartbeat transns
               0        0       0       0       2
```

* Pros: Even more metrics
* Cons: Output still not nice

More data can also be seen using the `-d` flag on all commands:

```
# ip -d l show dev wlp3s0
3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
    link/ether 00:aa:bb:cc:dd:cc brd ff:ff:ff:ff:ff:ff promiscuity 0 addrgenmode none numtxqueues 4 numrxqueues 1 gso_max_size 65536 gso_max_segs 6553

# ip -d r
unicast default via 192.168.2.1 dev wlp3s0 proto static scope global metric 600
unicast 192.168.2.0/24 dev wlp3s0 proto kernel scope link src 192.168.2.6 metric 600
```

* Pros: Even more data
* Cons: None at all

Finally, one last fancy feature. Let's watch for netlink messages in the kernel
while performing a DHCP request on an `UP` wireless interface:

```bash
# ip -d -t monitor
Timestamp: Fri May 25 16:39:14 2018 587277 usec
3: wlp3s0    inet 192.168.2.6/24 brd 192.168.2.255 scope global wlp3s0
       valid_lft forever preferred_lft forever
Timestamp: Fri May 25 16:39:14 2018 587313 usec
local 192.168.2.6 dev wlp3s0 table local proto kernel scope host src 192.168.2.6
Timestamp: Fri May 25 16:39:14 2018 587327 usec
broadcast 192.168.2.255 dev wlp3s0 table local proto kernel scope link src 192.168.2.6
Timestamp: Fri May 25 16:39:14 2018 587336 usec
unicast 192.168.2.0/24 dev wlp3s0 table main proto kernel scope link src 192.168.2.6
Timestamp: Fri May 25 16:39:14 2018 587345 usec
broadcast 192.168.2.0 dev wlp3s0 table local proto kernel scope link src 192.168.2.6
Timestamp: Fri May 25 16:39:14 2018 588963 usec
unicast default via 192.168.2.1 dev wlp3s0 table main proto boot scope global
Timestamp: Fri May 25 16:39:14 2018 597364 usec
192.168.2.1 dev wlp3s0 lladdr 11:11:11:ff:dd:bb REACHABLE
```


* Pros: Realtime overview of what's happening inside the kernel
* Cons: Output could be better


That's all for now. On my next post, I'll focus on `bridge` tool, which ships
as a part of `iproute2`. I don't think that you're still using that
unmaintained and deprecated tool called `brctl`, right? :)
