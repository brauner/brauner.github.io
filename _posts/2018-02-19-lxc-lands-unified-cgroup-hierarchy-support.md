---
layout: post
title: 'LXC Lands Unified cgroup Hierarchy Support'
---

![alt text](https://linuxcontainers.org/static/img/containers.png)

I'm excited to announce that we've recently added support the new unified
cgroup hierarchy (or cgroup v2, cgroup2) to LXC. This includes running system
containers that know about the unified cgroup hierarchy and application
containers that mostly won't really care about the cgroup layout. I'm not going
to do a deep dive into the differences between legacy cgroup hierarchies and
the unified cgroup hierarchy here. But if you're interested you can watch my
talk at last year's Container Camp Sydney

[![**Here be dragons**: some of what I say is likely invalid by now or I've come
up with simpler solutions.](http://img.youtube.com/vi/P6Xnm0IhiSo/0.jpg)](https://www.youtube.com/watch?v=P6Xnm0IhiSo)

### Currently existing cgroup layouts

But let's take a quick look at the different cgroup layouts out there.
Currently, there are three known cgroup layouts that a container runtime should
handle:

#### 1. Legacy cgroup hierarchies

This means that only legacy cgroup controllers are enabled and mounted into
either separate hierarchies or multiple individual cgroup hierarchies. The
mount layout of a legacy cgroup hierarchy has been standardized in recent
years. This is mainly due to the widespread use of systemd and its opinionated
way of how legacy cgroups should be mounted (Note, this is not a critique.).
A standard legacy cgroup layout will usually look like this:

```
├─/sys/fs/cgroup                    tmpfs  tmpfs  ro,nosuid,nodev,noexec,mode=755
│ ├─/sys/fs/cgroup/systemd          cgroup cgroup rw,nosuid,nodev,noexec,relatime,xattr,name=systemd
│ ├─/sys/fs/cgroup/devices          cgroup cgroup rw,nosuid,nodev,noexec,relatime,devices
│ ├─/sys/fs/cgroup/blkio            cgroup cgroup rw,nosuid,nodev,noexec,relatime,blkio
│ ├─/sys/fs/cgroup/cpu,cpuacct      cgroup cgroup rw,nosuid,nodev,noexec,relatime,cpu,cpuacct
│ ├─/sys/fs/cgroup/cpuset           cgroup cgroup rw,nosuid,nodev,noexec,relatime,cpuset,clone_children
│ ├─/sys/fs/cgroup/rdma             cgroup cgroup rw,nosuid,nodev,noexec,relatime,rdma
│ ├─/sys/fs/cgroup/hugetlb          cgroup cgroup rw,nosuid,nodev,noexec,relatime,hugetlb
│ ├─/sys/fs/cgroup/freezer          cgroup cgroup rw,nosuid,nodev,noexec,relatime,freezer
│ ├─/sys/fs/cgroup/net_cls,net_prio cgroup cgroup rw,nosuid,nodev,noexec,relatime,net_cls,net_prio
│ ├─/sys/fs/cgroup/perf_event       cgroup cgroup rw,nosuid,nodev,noexec,relatime,perf_event
│ ├─/sys/fs/cgroup/pids             cgroup cgroup rw,nosuid,nodev,noexec,relatime,pids
│ └─/sys/fs/cgroup/memory           cgroup cgroup rw,nosuid,nodev,noexec,relatime,memory
```

As you can see, most controllers (e.g. `devices`, `blkio`, `cpuset`) are
mounted into a separate cgroup hierarchy. They could be mounted differently but
given that this is how most userspace programs now mount cgroups and expect
cgroups to be mounted other forms of mounting them rarely need to be supported.

#### 2. Hybrid cgroup hierarchies

The mount layout of hybrid cgroup hierarchies is mostly identical to the mount
layout of the legacy cgroup hierarchies. The only difference usually being that
the unified cgroup hierarchy is mounted as well. The unified cgroup hierarchy
can easily be spotted by looking at the `FSTYPE` field in the output of the
findmnt command. For legacy cgroup hierarchies it will show cgroup as value and
for the unified cgroup hierarchy it will show cgroup2 as value. In the output
below the third field corresponds to the `FSTYPE`:

```
├─/sys/fs/cgroup                    tmpfs  tmpfs  ro,nosuid,nodev,noexec,mode=755
│ ├─/sys/fs/cgroup/unified          cgroup cgroup2 rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/cgroup/systemd          cgroup cgroup rw,nosuid,nodev,noexec,relatime,xattr,name=systemd
│ ├─/sys/fs/cgroup/devices          cgroup cgroup rw,nosuid,nodev,noexec,relatime,devices
│ ├─/sys/fs/cgroup/blkio            cgroup cgroup rw,nosuid,nodev,noexec,relatime,blkio
│ ├─/sys/fs/cgroup/cpu,cpuacct      cgroup cgroup rw,nosuid,nodev,noexec,relatime,cpu,cpuacct
│ ├─/sys/fs/cgroup/cpuset           cgroup cgroup rw,nosuid,nodev,noexec,relatime,cpuset,clone_children
│ ├─/sys/fs/cgroup/rdma             cgroup cgroup rw,nosuid,nodev,noexec,relatime,rdma
│ ├─/sys/fs/cgroup/hugetlb          cgroup cgroup rw,nosuid,nodev,noexec,relatime,hugetlb
│ ├─/sys/fs/cgroup/freezer          cgroup cgroup rw,nosuid,nodev,noexec,relatime,freezer
│ ├─/sys/fs/cgroup/net_cls,net_prio cgroup cgroup rw,nosuid,nodev,noexec,relatime,net_cls,net_prio
│ ├─/sys/fs/cgroup/perf_event       cgroup cgroup rw,nosuid,nodev,noexec,relatime,perf_event
│ ├─/sys/fs/cgroup/pids             cgroup cgroup rw,nosuid,nodev,noexec,relatime,pids
│ └─/sys/fs/cgroup/memory           cgroup cgroup rw,nosuid,nodev,noexec,relatime,memory
```

To be honest, this is not my favorite cgroup layout since it could potentially
mean that some cgroup controllers are mounted into separate legacy hierarchies
while others could be enabled in the unified cgroup hierarchy. That is not
difficult but annoying to handle cleanly. However, systemd usually plays nice
and only supports the empty unified cgroup hierarchy in hybrid cgroup layouts.

That is to say, all controllers are mounted into legacy cgroup hierarchies and
the unified hierarchy is just used by systemd to track processes, essentially
replacing the old named systemd legacy cgroup hierarchy.

#### 3. Unified cgroup hierarchy

The last option is to only mount the unified cgroup hierarchy direcly at
`/sys/fs/cgroup`:

```
├─/sys/fs/cgroup cgroup cgroup2 rw,nosuid,nodev,noexec,relatime
```

This will likely be the near future.

LXC in current git master will support all three layouts properly including
setting resource limits. So far, LXC has only provided the `lxc.cgroup.*`
namespace to set cgroup settings on legacy cgroup hierarchies. For example, to
set a limit on the number of cpus on the `cpuset` legacy cgroup hierarchy one
would simply specify `lxc.cgroup.cpuset.cpus = 1-2` in the containers config
file. The idea behind this is that the `lxc.cgroup.*` namespace simply takes
the name of a cgroup controller and a file that should be modified.

Similar to the `lxc.cgroup.*` legacy hierarchy namespace we have now introduced
the `lxc.cgroup2.*` namespace which follows the exact same logic but allows to
set cgroup limits on the unified hierarchy. This should allow users to easily
and intuitively transition from legacy cgroup layouts to unified cgroup layouts
in the near future if their distro of choice decides to do the switch.

One of the first benefactors will be [LXD](https://github.com/lxc/lxd) since we
have some users running on unified layouts. But of course, the feature is
available to all user of the API of the LXC shared library.
