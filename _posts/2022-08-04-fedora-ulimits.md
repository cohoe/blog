---
title: "Setting ulimits in a Fedora Systemd World"
date: 2022-08-04
published: true
categories: ulimits fedora linux systemd cinnamon
---

To set ulimits on a Systemd host: Don't bother with PAM/limits.conf and simply set a custom config in user.conf.d or system.conf.d.

Ulimits are resource limits imposed on processes and/or users. To do proper load testing on my workstation I needed to increase the
number of allowed open files (`NOFILE`). There are two values at play: Soft and Hard limits. Soft limit is the artificial cap imposed
by the kernel, but the user can kick it up to a maxmimum of a Hard limit. So for example you can temporarily kick up the amount of
`NOFILE`'s from 1024 with `ulimit -Sn 9001` but you cannot exceed a hard limit of 524288 (unless you're root).

Any good Google search will tell you how to achieve this through two methods: `limits.conf` and PAM, or via Systemd.

In Ye Olden Days(TM) we would set limits in `/etc/security/limits.conf` akin to:
```
#<domain>      <type>  <item>         <value>
#
grant          soft    nofile         9000
```

Then you had to do some PAM configuring to get these limits loaded when you logged in. But this is the world of tomorrow! We pray to our
one true lord and savior Systemd which handles limits through two contexts: System and User.

Defaults for Systemd are set in `/etc/systemd/system.conf` and `/etc/systemd/user.conf`. Each key takes a `soft[:hard]` value. I thought
I could set my limits here, but it never worked. I kept seeing the same limit of 1024. After much more Googling I stumbled across an
interesting command that would aggregate all configs in each context:
```
systemd-analyze cat-config /etc/systemd/user.conf
```

This aggregates all of the configs and their `conf.d`'s that I had no idea existed (oh and for fun aren't even in the same directory tree.
Lo and behold I found that Steam had installed its own limit files to `/usr/lib/systemd/user.conf.d/01-steam.conf` (and same path under system.conf.d).
Now I could have just deleted the files and been on my merry way, but deleting configs like this is bad practice because the package will often
just replace them at next update. This happens all the time in Apache-land and is why you should blank out default configs rather than eliminate them.
So the next best thing was to just insert my own `99-custom.conf` file in that directory with the values I needed:

```
[Manager]
DefaultLimitNOFILE=9001:1048576
```

A quick reboot and all is well.

