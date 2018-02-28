---
layout: post
title: 'On The Way To LXC 3.0: Moving The Cgroup Pam Module Into The LXC Tree (Including A Detour About Fully Unprivileged Containers)'
---

![alt text](https://linuxcontainers.org/static/img/containers.png)

Hey everyone,

This is another update about the development of `LXC 3.0`.

A few days ago the
[`pam_cgfs.so`](https://github.com/lxc/lxc/blob/master/src/lxc/pam/pam_cgfs.c)
pam module has been moved out of the [`LXCFS`](https://github.com/lxc/lxcfs)
tree and into the [`LXC`](https://github.com/lxc/lxc) tree.
This means `LXC 3.0` will be shipping with `pam_cgfs.so` included. The pam
module has been placed under the `configure.ac` flags `--enable-pam` and
`--disable-pam`. By default `pam_cgfs.so` is disabled. Distros that are
currently shipping `pam_cgfs.so` through `LXCFS` should adapt their packaging
accordingly and pass `--enable-pam` during the `configure` stage of `LXC`.

#### What's That `pam_cgfs.so` Pam Module Again?

Let's take short detour ("short" *cough* *cough*). **`LXC` has supported fully
unprivileged containers since `2013` when `user namespace` support was merged
into the kernel. (/me tips hat to Serge Hallyn and Eric Biedermann).** Fully
unprivileged containers are containers using `user namespaces` and idmappings
which are run by normal (non-root) users. But let's not talk about this let's
show it. The first asciicast shows a fully unprivileged system container
running with a rather complex idmapping in a new user namespace:

[![asciicast](https://asciinema.org/a/166127.png)](https://asciinema.org/a/166127)

The second asciicast shows a fully unprivileged application container **running
without a mapping for root inside the container**. In fact, it runs with just
a single idmap that maps my own `host uid` `1000` and `host gid` `1000` to
`container uid` `1000` and `container gid` `1000`. Something which I can do
without requiring any privilege at all. We've been doing this a long time at
`LXC`:

[![asciicast](https://asciinema.org/a/155311.png)](https://asciinema.org/a/155311)

As you can see no non-standard privileges are used when setting up and running
such containers. In fact, you could remove even the standard privileges all
unprivileged users have available through standard system tools like
`newuidmap` and `newgidmap` to setup idmappings (This is what you see in the
second asciicast.). But this comes at a price, namely that cgroup management is
not available for fully unprivileged containers. But we at `LXC` want you to
be able to restrict the containers your run in the same way that the system
administrator wants to restrict unprivileged users themselves. This is just
good practice to prevent excessive resource consumption. What this means is
that you should be free to delegate resources that you have been given by the
system administrator to containers. This e.g. allows you to limit the cpu usage
of the container, or the number of processes it is allowed to spawn, or the
memory it is allowed to consume. But unprivileged cgroup management is not
easily possible with most init system. That's why the `LXC` team came up with
`pam_cgfs.so` a long time ago to make things easier. In essence, the
`pam_cgfs.so` pam module takes care of placing unprivileged users into writable
cgroups at login. The cgroups that are supposed to be writable can be specified
in the corresponding pam configuration file for your distro (probably something
under `/etc/pam.d`). For example, if you wanted your user to be placed into
a writable cgroup for all enabled cgroup hierarchies you could specify `all`:

```
session	optional	pam_cgfs.so -c all
```

If you only want your user to be placed into writable cgroups for the
`freezer`, `memory`, `unified` and the named `systemd` hierarchy you would
specify:

```
session	optional	pam_cgfs.so -c freezer,memory,name=systemd,unified
```

This would lead `pam_cgfs.so` to create the common cgroup `user` and also
create a cgroup just for my own user in there. For example, my user is called
`chb`. This would cause `pam_cgfs.so` to create the
`/sys/fs/cgroup/freezer/user/chb/0` inside the `freezer` hierarchy. If
`pam_cgfs.so` finds that your init system has already placed your users inside
a session specific cgroup it will be smart enough to detect it and re-use that
cgroup. This is e.g. the case for the named `systemd` cgroup hierarchy.

```
chb@conventiont|~
> cat /proc/self/cgroup
12:hugetlb:/
11:devices:/user.slice
10:memory:/user.slice
9:perf_event:/
8:net_cls,net_prio:/
7:cpu,cpuacct:/user.slice
6:rdma:/
5:pids:/user.slice/user-1000.slice/session-1.scope
4:cpuset:/
3:blkio:/user.slice
2:freezer:/user/chb/0
1:name=systemd:/user.slice/user-1000.slice/session-1.scope
0::/user.slice/user-1000.slice/session-1.scope
```

Christian
