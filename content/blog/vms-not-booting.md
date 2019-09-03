---
Title: VMs available in >5 minutes after upgrading Debian. Why?
date: 2019-09-03
---

At [work](https://www.skroutz.gr), we manage a fleet of bare metal servers and
virtual machines, running Debian GNU/Linux. Upgrades to a new major are often
very challenging; not because of the actual upgrade path, but because of the
unwanted side-effects that noone had predicted.

One of these side-effects was: "Why this shiny Buster VM takes 5 minutes to
boot?". Together with my colleague
[Alexandros Afentoulis](https://alexaf.gitlab.io/), we figured out the issue,
fixed it and wrote a post about it. The real root cause is a number of changes
that happened to Debian GNU/Linux regarding entropy gathering (!).

You can find our work on Alexandros' blog
[here](https://alexaf.gitlab.io/posts/linux_early_entropy_starvation_buster/).
