---
layout: post
title: 'On The Way To LXC 3.0: Removal of cgmanager And cgfs cgroup Drivers'
---

Hey everyone,

This is another update about the development of `LXC 3.0`. As of yesterday the
`cgmanager` and `cgfs` cgroup drivers have been removed from the codebase. In
the good long tradition of all `LXC` projects to try our hardest to never
regress our users and to clearly communicate invasive changes I wanted to take
the time to explain why these two cgroup drivers have been removed in a little
more detail.

### [cgmanager](https://linuxcontainers.org/cgmanager/introduction/)
The `cgmanager` cgroup driver relies on the upstream `cgmanager` project which
as created and written by fellow `LXC` and `LXD` maintainer Serge
Hallyn. In a time where proper userspace cgroup management was still an
unsolved problem `cgmanager` was a stroke of genius and solved a lot of issues.
The most prominent ones where:
- using cgroups when nesting containers
- enabling cgroup management for unprivileged users running unprivileged
  containers, i.e. containers emplying user namespaces and idmappings.

To this day it isn't possible to do unprivileged cgroup management. The
upstream maintainers are opinionated in that respect and probably are
rightly so. To delegate cgroups to unprivileged users privilege is
required. To address this problem the `cgmanager` driver was bundled with
the `cgmanager` daemon.
The `cgmanager` daemon and it's driver counterpart in `LXC` were an
appropriate solution in a time where init systems did not do any proper
cgroup managment. With the rise of `systemd` this changed. Nowadays,
`systemd` is effectively its own cgroup manager. While this didn't
necessarily had to mean the end of cgmanager it seemed unlikely that users
would run two competing cgroup managers on the system. (To be honest, one
can still consider it controversial whether `systemd` in addition to using
a dedicated cgroup or cgroups to do process tracking should also be a
full cgroup manager. But it is at least an understandable choice.)
Additionally, a new cgroup driver was added to `LXC` that did not rely on
a cgroup manager daemon. It was written to be compatible with
unprivileged cgroup management and cgroup management when nesting
containers. (Unprivileged cgroup management is usually done by employing
our pam module which places users in writable cgroups on login.) While
unprivileged cgroup management is still not a solved problem, especially
with the rise of the unified cgroup hierarchy, `cgmanager` has outlived its
purpose. It has been marked deprecated for a long long time now. So
while it is still supported on the `LXC 1.*` stable branch it is now officially
dead. A big thanks to Serge for coming up with this project!

#### [cgfs](https://github.com/lxc/lxc/commit/1a8848b371cf2c86400f58fc64bf7ecc2cf5b261)
The `cgfs` driver, if I have my facts straight, stems from a time even
before `cgmanager` was a thing. Actually, it is so old that mounting
cgroups was not yet clearly standardized. For example, in these (dark)
times some distros would wount cgroups at `/dev/cgroup` while others would
mount them at `/sys/fs/cgroup`. In addition some distros would mount all
controllers into a single hierarchy located directly at `/sys/fs/cgroup`
other distros would mount some controllers into separate hierarchies and
others together. Over time the upstream cgroup maintainers became more
and more opinionated how and where cgroups should be mounted. At the
same time `systemd` decided to mount cgroups in a very specific layout.
This layout has been the default on nearly all distros that either use
`systemd` or have their init system mount cgroups.
The `cgfs` driver tried to accommodate **all** possible layouts, i.e. it
tried to find the mountpoint for each controller and mimic this layout
for the container. It obviously came with a lof of flexibility and the
code showed it. It was a massive `C` file that did a lot of complex
parsing. Nowadays, it is perfectly reasonable for a performant cgroup
driver to make simplifying assumptions about how cgroups are to be
mounted. This is especially true with the unified cgroup hierarchy.
Being able to rely on these standards is a good thing. It means code can
be simplified, logic can be simplified, and special cases become less
likely. In the face of these changes, carrying the `cgfs` driver into the
future would have meant accepting a significant but unnecessary
performance penalty that outlived its own cause. That can't be a good
thing. It also would have meant keeping around massively complex
codepaths that would never really be hit or only hit in very rare
circumstances on exotic systems.

#### More General Reasons
These are some arguments that apply to both cgroup drivers and to
coding in general:
- **code blindness**

  This is a general phenomenon that comes in two forms. Most developers
  will know what I'm talking about. It either means ones has stared at a
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

#### Adding New Cgroup Drivers
This is a note to developers more than to users. The current cgroup
driver is not the last word. It has been adapted to be compatible with
legacy cgroup hierarchies and the unified cgroup hierarchies and
actually also supports hybrid cgroup layouts where some controllers are
mounted into separate legacy cgroup hierarchies and others are present
in the unified cgroup hierarchy. But if there is a legitimate need for
someone to come up with a different cgroup driver they should be aware
that the way the `LXC` cgroup drivers are written is modular. In essence
it is close to what some languages would call an "interface". New cgroup
drivers just need to implement it. :)

Removing code that has been around and was maintained for a long time is
of course hard since a lot of effort and thought has gone into it and
one needs to be especially careful to not regress users that still rely
on such codepaths quite heavily. But mostly it is a sign of a healthy
project: withing a week `LXC` got rid of `4,479` lines of code.

At the end, I have want to say thank you to my fellow maintainers [Stéphane
Graber][1] ([@stgraber](https://twitter.com/stgraber)) on and [Serge Hallyn][2]
([@sehh](https://twitter.com/sehh)) for all the reviews, merges, ideas, and
constructive criticism over all those months. And of course, thanks to all the
various contributors be it from companies like Huawei, Nvidia, Red Hat or
individual contributors sending fixes all over the place. We greatly appreciate
this! Keep the patches coming.

Christian

[1]: https://twitter.com/stgraber
[1]: https://stgraber.org/
[2]: https://twitter.com/sehh 
[2]: https://s3hh.wordpress.com/
