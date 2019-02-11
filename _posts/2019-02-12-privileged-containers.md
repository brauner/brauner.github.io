---
layout: post
title: 'Runtimes And the Curse of the Privileged Container'
---

#### Introduction ([CVE-2019-5736][2])

Today, Monday, 2019-02-11, 14:00:00 CET [CVE-2019-5736][2] was released:

> The vulnerability allows a malicious container to (with minimal user
> interaction) overwrite the host runc binary and thus gain root-level
> code execution on the host. The level of user interaction is being able
> to run any command (it doesn't matter if the command is not
> attacker-controlled) as root within a container in either of these
> contexts:
>
>   * Creating a new container using an attacker-controlled image.
>   * Attaching (docker exec) into an existing container which the
>     attacker had previous write access to.

I've been working on a fix for this issue over the last couple of weeks
together with [Aleksa][1] a friend of mine and maintainer of runC. When he
notified me about the issue in runC we tried to come up with an exploit for
[LXC][3] as well and though harder it is doable.
I was interested in the issue for technical reasons and figuring out how to
reliably fix it was quite fun (with a proper dose of pure hatred). It also
caused me to finally write down some personal thoughts I had for a long time
about how we are running containers.

#### What are Privileged Containers?

At a first glance this is a question that is probably trivial to anyone who has
a decent low-level understanding of containers. Maybe even most users by now
will know what a privileged container is. A first pass at defining it would be
to say that a privileged container is a container that is owned by root.
Looking closer this seems an insufficient definition. What about containers
using user namespaces that are started as root?
It seems we need to distinguish between what ids a container is running with.
So we could say a privileged container is a container that is running as root.
However, this is still wrong. Because "running as root" can either be seen as
meaning "running as root as seen from the outside" or "running as root from the
inside" where "outside" means "as seen from a task outside the container" and
"inside" means "as seen from a task inside the container".

What we really mean by a privileged container is a container where the
semantics for id 0 are the same inside and outside of the container ceteris
paribus. I say "ceteris paribus" because using LSMs, seccomp or any other
security mechanism will not cause a change in the meaning of id 0 inside and
outside the container. For example, a breakout caused by a bug in the runtime
implementation will give you root access on the host.

An unprivileged container then simply is any container in which the semantics
for id 0 inside the container are different from id 0 outside the container.
For example, a breakout caused by a bug in the runtime implementation will not
give you root access on the host by default. This should only be possible if
the kernel's user namespace implementation has a bug.

The reason why I like to define privileged containers this way is that it also
lets us handle edge cases. Specifically, the case where a container is using
a user namespace but a hole is punched into the idmapping at id 0 aka where id
0 is mapped through. Consider a container that uses the following idmappings:
```
id: 0 100000 100000
```
This instructs the kernel to setup the following mapping:
```
id: container_id(0) -> host_id(100000)
id: container_id(1) -> host_id(100001)
id: container_id(2) -> host_id(100002)
.
.
.

container_id(100000) -> host_id(200000)
```
With this mapping `container_id(0) != host_id(100000)`. But now consider
the following mapping:
```
id: 0 0 1
id: 1 100001 99999
```
This instructs the kernel to setup the following mapping:
```
id: container_id(0) -> host_id(0)
id: container_id(1) -> host_id(100001)
id: container_id(2) -> host_id(100002)
.
.
.

container_id(99999) -> host_id(199999)
```
In contrast to the first example this has the consequence that `container_id(0)
== host_id(0)`.
I would argue that any container that at least punches a hole for id 0 into its
idmapping up to specifying an identity mapping is to be considered a privileged
container.

As a sidenote, Docker containers run as privileged containers by default. There
is usually some confusion where people think because they do not use the
`--privileged` flag that Docker containers run unprivileged. This is wrong.
What the `--privileged` flag does is to give you even more permissions by e.g.
not dropping (specific or even any) capabilities. One could say that such
containers are almost "super-privileged".

#### The Trouble with Privileged Containers

The problem I see with privileged containers is essentially captured by
[LXC][2]'s and [LXD][4]'s upstream security position which we have held since
at least [2015][8] but probably even earlier. I'm quoting from our [notes about
privileged containers][6]:

> Privileged containers are defined as any container where the container uid 0 is
> mapped to the host's uid 0. In such containers, protection of the host and
> prevention of escape is entirely done through Mandatory Access Control
> (apparmor, selinux), seccomp filters, dropping of capabilities and namespaces.
> 
> Those technologies combined will typically prevent any accidental damage of the
> host, where damage is defined as things like reconfiguring host hardware,
> reconfiguring the host kernel or accessing the host filesystem.
> 
> LXC upstream's position is that those containers aren't and cannot be
> root-safe.
> 
> They are still valuable in an environment where you are running trusted
> workloads or where no untrusted task is running as root in the container.
> 
> We are aware of a number of exploits which will let you escape such containers
> and get full root privileges on the host. Some of those exploits can be
> trivially blocked and so we do update our different policies once made aware of
> them. Some others aren't blockable as they would require blocking so many core
> features that the average container would become completely unusable.
> 

[...]

> As privileged containers are considered unsafe, we typically will not consider
> new container escape exploits to be security issues worthy of a CVE and quick
> fix. We will however try to mitigate those issues so that accidental damage to
> the host is prevented.

LXC's upstream position for a long time has been that privileged containers are
not and cannot be root safe. For something to be considered root safe it should
be safe to hand root access to third parties or tasks.

#### Running Untrusted Workloads in Privileged Containers

is insane. That's about everything that this paragraph should contain. The fact
that the semantics for id 0 inside and outside the container are identical
entails that any meaningful container escape will have the attacker gain root
on the host.

#### [CVE-2019-5736][2] Is a Very Very Very Bad Privilege Escalation to Host Root

[CVE-2019-5736][2] is an excellent illustration of such an attack. Think about
it: a process running **inside** a privileged container can rather trivially
corrupt the binary that is used to attach to the container. This allows an
attacker to create a custom ELF binary on the host. That binary could do
anything it wants:
- could just be a binary that calls `poweroff`
- could be a binary that spawns a root shell
- could be a binary that kills other containers when called again to attach
- could be `suid` `cat`
- .
- .
- .

The attack vector is actually slightly worse for runC due to its architecture.
Since runC exits after spawning the container it can also be attacked through
a malicious container image. Which is super bad given that a lot of container
workload workflows rely on downloading images from the web.

[LXC][2] cannot be attacked through a malicious image since the monitor process
(a singleton per-container) never exits during the containers life cycle. Since
the kernel does not allow modifications to running binaries it is not possible
for the attacker to corrupt it. When the container is shutdown or killed the
attacking task will be killed before it can do any harm. Only when the last
process running inside the container has exited will the monitor itself exit.
This has the consequence, that if you run privileged OCI containers via our
`oci` template with [LXC][2] your are not vulnerable to malicious images. Only
the vector through the attaching binary still applies.

#### The Lie that Privileged Containers can be safe

We as upstream for [LXC][2] and [LXD][4] have not taken this position that
privileged containers are not and cannot be root safe for fun. We have taken
that position because privileged containers are inherently not suited to run
untrusted workloads and anyone who is advertising that this can be done is
wrong!

#### Unprivileged Containers as Default

As upstream for [LXC][2] and [LXD][4] we have been advocating the use of
unprivileged containers by default for years. Way ahead before anyone else did.
Our low-level library [LXC][2] has supported unprivileged containers since 2013
when user namespaces were merged into the kernel. With [LXD][4] we have taken
it one step further and made unprivileged containers the default and privileged
containers opt-in for that very matter: privileged containers aren't safe. We
even allow you to have per-container idmappings to make sure that not just each
container is isolated from the host but also all containers from each other.

For years we have been advocating for unprivileged containers on conferences,
in blogposts, and whenever we have spoken to people but somehow this whole
industry has chosen to rely on privileged containers.

The good news is that we are seeing changes as people become more familiar with
the perils of privileged containers. Let this recent CVE be another reminder
that unprivileged containers need to be the default.

#### Are LXC and LXD affected?

I have seen this question asked all over the place so I guess I should add
a section about this too:

- Unprivileged [LXC][2] and [LXD][3] containers are not affected.

- Any privileged [LXC][2] and [LXD][3] container running on a read-only rootfs
  is not affected.

- Privileged [LXC][2] containers in the definition provided above are affected.
  Though the attack is more difficult than for runC. The reason for this is
  that the `lxc-attach` binary does not exit before the program in the
  container has finished executing. This means an attacker would need to open
  an `O_PATH` file descriptor to `/proc/self/exe`, `fork()` itself into the
  background and re-open the `O_PATH` file descriptor through
  `/proc/self/fd/<O_PATH-nr>` in a loop as `O_WRONLY` and keep trying to write
  to the binary until such time as `lxc-attach` exits. Before that it will not
  succeed since the kernel will not allow modification of a running binary.

- Privileged [LXD][3] containers are only affected if the daemon is restarted
  other than for upgrade reasons. This should basically never happen.
  The [LXD][3] daemon never exits so any write will fail because the kernel
  does not allow modification of a running binary.
  If the [LXD][3] daemon is restarted because of an upgrade the binary will be
  swapped out and the file descriptor used for the attack will write to the old
  in-memory binary and not to the new binary.

#### Chromebooks with Crostini using LXD are not affected

Chromebooks use [LXD][3] as their default container runtime are not affected.
First of all, all binaries reside on a read-only filesystem and second,
[LXD][3] does not allow running privileged containers on Chromebooks through
the `LXD_UNPRIVILEGED_ONLY` flag. For more details see this [link][7].

#### Fixing CVE-2019-5736

To prevent this attack, [LXC][2] has been patched to create a temporary copy of
the calling binary itself when it attaches to containers (cf.
[6400238d08cdf1ca20d49bafb85f4e224348bf9d][9]). To do this [LXC][2] can be
instructed to create an anonymous, in-memory file using the `memfd_create()`
system call and to copy itself into the temporary in-memory file, which is then
sealed to prevent further modifications. [LXC][2] then executes this sealed,
in-memory file instead of the original on-disk binary. Any compromising write
operations from a privileged container to the host [LXC][2] binary will then
write to the temporary in-memory binary and not to the host binary on-disk,
preserving the integrity of the host [LXC][2] binary. Also as the temporary,
in-memory [LXC][2] binary is sealed, writes to this will also fail. To not
break downstream users of the shared library this is opt-in by setting
`LXC_MEMFD_REXEC` in the environment. For our `lxc-attach` binary which is the
only attack vector this is now done by default.

Workloads that place the [LXC][2] binaries on a read-only filesystem or prevent
running privileged containers can disable this feature by passing
`--disable-memfd-rexec` during the `configure` stage when compiling [LXC][2].

[1]: https://www.cyphar.com/
[2]: https://seclists.org/oss-sec/2019/q1/119
[3]: https://github.com/lxc/lxc
[4]: https://github.com/lxc/lxd
[5]: https://github.com/lxc/lxd
[6]: https://linuxcontainers.org/lxc/security/#privileged-containers
[7]: https://www.reddit.com/r/Crostini/comments/apkz8t/crostini_containers_likely_vulnerable_to/
[8]: https://github.com/lxc/linuxcontainers.org/commit/b1a45aef6abc885594aab2ce6bdeb2186c5e0973
[9]: https://github.com/lxc/lxc/commit/6400238d08cdf1ca20d49bafb85f4e224348bf9d
