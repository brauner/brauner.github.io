---
layout: post
title: 'On The Way To LXC 3.0: Removal of cgmanager And cgfs cgroup Drivers'
---

![alt text](https://linuxcontainers.org/static/img/containers.png)

Hey everyone,

This is another update about the development of `LXC 3.0`.

As of yesterday the `cgmanager` and `cgfs` cgroup drivers have been
removed from the codebase. In the good long tradition of all `LXC`
projects to try our hardest to never regress our users and to clearly
communicate invasive changes I wanted to take the time to explain why
these two cgroup drivers have been removed in a little more detail.

### [CGManager](https://linuxcontainers.org/cgmanager/introduction/)
The `CGManager` cgroup driver relies on the upstream `CGManager` project which
was created and written by fellow `LXC` and `LXD` maintainer Serge
Hallyn back in late 2013.

The need for `CGManager` has been fading over the years as its main
features can now be achieved in more standard and efficient ways:
- Allowing nested containers to control their own cgroups.
- Enabling cgroup management for unprivileged users running unprivileged
  containers, i.e. containers employing user namespaces and idmappings.

A first effort to deprecate CGManager happened with the inclusion of the
new `cgfsng` cgroup driver in LXC combined with `LXCFS` support for
creating a per-container cgroup view in userspace.

The `LXCFS` approach had the benefit of working with all existing software
that would normally interact with cgroups through the filesystem and was
also more efficient (multi-threaded) compared to the single-threaded
`DBUS` API that `CGManager` was offering.

The later inclusion of the cgroup namespace in the mainline kernel
finally moved all of this into the kernel, completely removing the need
for a userspace solution to the problem.

`CGManager` itself is currently considered as deprecated and will not see
any further release and so there is little point in `LXC` keeping support
for it.

#### [cgfs](https://github.com/lxc/lxc/commit/1a8848b371cf2c86400f58fc64bf7ecc2cf5b261)
The `cgfs` driver dates back from the origins of the cgroup subsystem
and early integration in Linux distributions.

At that point in time, it was somewhat common for cgroup controllers to
all be co-mounted or be co-mounted in big chunks, often under
`/dev/cgroup` or `/cgroup`.

`LXC` therefore needed a lot of logic to figure out exactly what cgroup
controller could be found and where. It also had to enable a number of
different flags to have the then widely different controllers behave in
a similar way.

Nowadays, all Linux distributions that setup cgroups will mount a split
layout, typically with one controller per directory under
`/sys/fs/cgroup`. `LXC` can rely on this and so knows exactly where to find
all cgroup controllers without having to do complex mount table parsing
and guesses.

That's what the `cgfsng` driver, introduced in `LXC 2.0`, does with the old
`cgfs` driver only there as a fallback. We've very rarely witnessed that
`LXC` fell back to the `cgfs` driver and if it did it usually resulted in
another failure.

As the `cgfs` driver is old, complex and hard to maintain and doesn't
seem to actually be handling any real world use cases, it will similarly
be dropped.

#### More General Reasons
These are some arguments that apply to both cgroup drivers and to
coding in general:
- **code blindness**

  This is a general phenomenon that comes in two forms. Most developers
  will know what I'm talking about. It either means one has stared at a
  codepath for too long to really see problems anymore or it means that
  one has literally forgotten that a codepath actually exists. The
  latter is what I fear would happen with the `cgfs` driver. One day, I'd
  be looking at that code muttering "I have no memory of this place.".
- **forgotten logic**

  Reasoning about fallback/legacy codepaths is not a thing developers
  will have to do often. This makes it less likely for them to figure
  out bugs quickly. This is frustrating to users but also to developers
  since it increases maintenance costs significantly.

- **a well of bugs**

  Legacy codepaths are a good source for bugs. This is especially true
  for the cgroup codepaths because the rise of the unified cgroup
  hierarchy changes things significantly. A lot of assumptions about
  cgroup management are changing and updating all three cgroup drivers
  would be a massive undertaking and actually pointless.

#### Adding New cgroup Drivers
This is a note to developers more than to users. The [current `cgfsng` cgroup
driver](https://github.com/lxc/lxc/blob/master/src/lxc/cgroups/cgfsng.c) is not
the last word. It has been adapted to be compatible with legacy cgroup
hierarchies and the unified cgroup hierarchies and actually also supports
hybrid cgroup layouts where some controllers are mounted into separate legacy
cgroup hierarchies and others are present in the unified cgroup hierarchy. But
if there is a legitimate need for someone to come up with a different cgroup
driver they should be aware that the way the `LXC` cgroup drivers are written
is modular. In essence it is close to what some languages would call an
"interface". New cgroup drivers just need to implement it. :)

Removing code that has been around and was maintained for a long time is
of course hard since a lot of effort and thought has gone into it and
one needs to be especially careful to not regress users that still rely
on such codepaths quite heavily. But mostly it is a sign of a healthy
project: within a week `LXC` got rid of `4,479` lines of code.

At the end, I want to say thank you to my fellow maintainers [Stéphane
Graber][1] ([@stgraber](https://twitter.com/stgraber)) and [Serge Hallyn][2]
([@sehh](https://twitter.com/sehh)) for all the reviews, merges, ideas, and
constructive criticism over all those months. And of course, thanks to all the
various contributors be it from companies like Huawei, Nvidia, Red Hat or
individual contributors sending fixes all over the place. We greatly appreciate
this! Keep the patches coming.

Christian & Stéphane

[1]: https://twitter.com/stgraber
[1]: https://stgraber.org/
[2]: https://twitter.com/sehh 
[2]: https://s3hh.wordpress.com/
