---
title: "A performance story: How a faulty QSFP crippled a Ceph cluster"
date: "2018-08-30"
---

At [work](https://grnet.gr), we encountered a strange case of performance
degradation on one of our production [Ceph](https://ceph.com) clusters
which provides storage to our public cloud
[~okeanos](https://okeanos.grnet.gr).

We wrote a post at our [tech blog](https://blog.noc.grnet.gr), explaining
the problem, our debugging journey, how we solved it and what we've done
for the future. Long story short, a QSFP in the underlying network
infrastructure, introduced huge packet loss and caused performance
issues to our cloud. Of course, it was not Ceph's fault :).

The post can be found
[here](https://blog.noc.grnet.gr/2018/08/29/a-performance-story-how-a-faulty-qsfp-crippled-a-whole-ceph-cluster/).
