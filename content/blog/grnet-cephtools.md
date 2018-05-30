---
title: Open sourcing GRNET's Ceph tools
date: 2018-05-30
---

I'm not afraid to say that I really like [Ceph](https://ceph.com/). It's one of
the first distributed systems that I got in touch with and I really enjoy
working and providing useful services with it. But as all large and complex
systems, it tends to get quickly difficult to perform operations on it. For
that reason, my team invested some time and wrote a bunch of useful tools that
we use in order to manage our Ceph clusters. 

I'm happy to announce these tools are open-sourced by GRNET and can be found
on [Github](https://github.com/grnet/cephtools). For now, this repo includes
check_mk health checks for Luminous, some Ansible playbooks and a lot of
scripts that we used while provisioning our new clusters. We also informed
[ceph-users](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2018-May/026456.html)
about it, in case someone will find these tools useful.

For the record, this repo is my first piece of Free Software :)

Take a look if interested!
