# The Seccomp Notifier - New Frontiers in Unprivileged Container Development

#### Introduction

As most people know by know we do a lot of upstream kernel development.  This stretches over multiple areas and of course we also do a lot of kernel work around containers.  In this article I'd like to take a closer look at the new seccomp notify feature we have been developing both in the kernel and in userspace and that is seeing more and more users.  I've talked about this feature quite a few times at various conferences (just recently again at [OSS NA](https://ossna2020.sched.com/event/c3WE/making-unprivileged-containers-more-useable-christian-brauner-canonical])) over the last two years but never actually sat down to write a blogpost about it.  This is something I had wanted to do for quite some time. First, because it is a very exciting feature from a purely technical perspective but also from the new possibilities it opens up for (unprivileged) containers and other use-cases.

#### The Limits of Unprivileged Containers

That (Linux) Containers are a userspace fiction is a well-known dictum nowadays.  It simply expresses the fact that there is no container kernel object in the Linux kernel.  Instead, userspace is relatively free to define what a container is.  But for the most part userspace agrees that a container is somehow concerned with isolating a task or a task tree from the host system.  This is achieved by combining a multitude of Linux kernel features.  One of the better known kernel features that is used to build containers are namespaces.  The number of namespaces the kernel supports has grown over time and we are currently at eight.  Before you go and look them up on `namespaces(7)` here they are:

- cgroup: `cgroup_namespaces(7)`
- ipc: `ipc_namespaces(7)`
- network: `network_namespaces(7)`
- mount: `mount_namespaces(7)`
- pid: `pid_namespaces(7)`
- time: `time_namespaces(7)`
- user: `user_namespaces(7)`
- uts: `uts_namespaces(7)`

Of these eight namespaces the user namespace is the only one concerned with isolating core privilege concepts on Linux such as user- and group ids, and capabilities.

Quite often we see tasks in userspace that check whether they run as root or whether they have a specific capability (e.g. `CAP_MKNOD` is required to create device nodes) and it seems that when the answer is "yes" then the task is actually a privileged task.  But as usual things aren't that simple.  What the task thinks it's checking for and what the kernel really is checking for are possibly two very different things.  A naive task, i.e. a task not aware of user namespaces, might think it's asking whether it is privileged with respect to the whole system aka the host but what the kernel really checks for is whether the task has the necessary privileges relative to the user namespace it is located in.

In most cases the kernel will not check whether the task is privileged with respect to the whole system.  Instead, it will almost always call a function called `ns_capable()` which is the kernel's way of checking whether the calling task has privilege in its current user namespace.

For example, when a new user namespace is created by setting the `CLONE_NEWUSER` flag in `unshare(2)` or in `clone3(2)` the kernel will grant a full set of capabilities to the task that called `unshare(2)` or the newly created child task via `clone3(2)` _within_ the new user namespace.  When this task now e.g. checks whether it has the `CAP_MKNOD` capability the kernel will report back that it indeed has that capability.  The key point though is that this "yes" is not a global "yes", i.e. the question "Am I privileged enough to perform this operation?" only applies to the current user namespace (and technically any nested user namespaces) not the host itself.

This distinction is important when  trying to understand why a task running as root in a new user namespace with all capabilities raised will still see `EPERM` when e.g. trying to call `mknod("/dev/mem", makedev(1, 1))` even though it seems to have all necessary privileges.  The reason for this counterintuitive behavior is that the kernel isn't always checking whether you are privileged against your current user namespace.  Instead, for any operation that it thinks is dangerous to expose to unprivileged users it will check whether the task is privileged in the initial user namespace, i.e. the host's user namespace.

Creating device nodes is one such example: if a task running in a user namespace were to be able to create character or block device nodes it could e.g. create `/dev/kmem` or any other critical device and use the device to take over the host.  So the kernel simply blocks creating all device nodes in user namespaces by always performing the check for required privileges against the initial user namespace.  This is of course technically inconsistent since capabilities are per user namespace as we observed above.

Other examples where the kernel requires privileges in the initial user namespace are mounting of block devices.  So simply making a disk device node available to an unprivileged container will still not make it useable since it cannot mount it.  On the other hand, some filesystems like `cgroup`, `cgroup2`, `tmpfs`, `proc`, `sysfs`, and `fuse` can be mounted in user namespace (with some caveats for `proc` and `sys` but we're ignoring those details for now) because the kernel can guarantee that this is safe.

But of course these restrictions are annoying.  Not being able to mount block devices or create device nodes means quite a few workloads are not able to run in containers even though they could be made to run safely.  Quite often a container manager like `LXD` will know better than the kernel when an operation that a container tries to perform is safe.

A good example are device nodes.  Most containers bind-mount the set of standard devices into the container otherwise it would not work correctly:

```
/dev/console
/dev/full
/dev/null
/dev/random
/dev/tty
/dev/urandom
/dev/zero
```

Allowing a container to create these devices would be safe.  Of course, the container will simply bind-mount these devices during container startup into the container so this isn't really a serious problem.  But any program running inside the container that wants to create these harmless devices nodes would fail.

The other example that was mentioned earlier is mounting of block-based filesystems.  Our users often instruct LXD to make certain disk devices available to their containers because they know that it is safe.  For example, they could have a dedicated disk for the container or they want to share data with or among containers.  But the container could not mount any of those disks.

For any use-case where the administrator is aware that a device node or disk device is missing from the container LXD provides the ability to hotplug them into one or multiple containers.  For example, here is how you'd hotplug `/dev/zero` into a running container:

```
 brauner@wittgenstein|~
> lxc exec f5 -- ls -al /my/zero

brauner@wittgenstein|~
> lxc config device add f5 zero-device unix-char source=/dev/zero path=/my/zero
Device zero-device added to f5

brauner@wittgenstein|~
> lxc exec f5 -- ls -al /my/zero
crw-rw---- 1 root root 1, 5 Jul 23 10:47 /my/zero
```

But of course, that doesn't help at all when a random application inside the container calls `mknod(2)` itself.  In these cases LXD has no way of helping the application by hotplugging the device as it's unaware that a mknod syscall has been performed.

So the root of the problem seems to be:
- A task inside the container performs a syscall that will fail.
- The syscall would not need to fail since the container manager knows that it is safe.
- The container manager has no way of knowing when such a syscall is performed.
- Even if the the container manager would know when such a syscall is performed it has no way of inspecting it in detail.

So a potential solution to this problem seems to be to enable the container manager or any sufficiently privileged task to take action on behalf of the container whenever it performs a syscall that would usually fail.  So somehow we need to be able to interact with the syscalls of another task.

#### Seccomp - The Basics of Syscall Interception

The obvious candidate to look at is seccomp.  Short for "secure computing" it provides a way of restricting the syscalls of a task either by allowing only a subset of the syscalls the kernel supports or by denying a set of syscalls it thinks would be unsafe for the task in question.  But seccomp allows even more advanced configurations through so-called "filters".  Filters are BPF programs (Not to be equated with eBPF. BPF is a predecessor of eBPF.) that can be written in userspace and loaded into the kernel.  For example, a task could use a seccomp filter to only allow the `mount()` syscall and only those mount syscalls that create bind mounts.  This simple syscall management mechanism has made seccomp an essential security feature for a lot of userspace programs.  Nowadays it is considered good practice to restrict any critical programs to only those syscalls it absolutely needs to run successfully.  Browser-based sandboxes and containers being prime examples but even systemd services can be seccomp restricted.

At its core seccomp is nothing but a syscall interception mechanism.  One way or another every operating system has something that is at least roughly comparable.  The way seccomp works is that it intercepts syscalls right in the architecture specific syscall entry paths.  So the seccomp invocations themselves live in the architecture specific codepaths although most of the logical around it is architecture agnostic.

Usually, when a syscall is performed, and no seccomp filter has been applied to the task issuing the syscall the kernel will simply lookup the syscall number in the architecture specific syscall table and if it is a known syscall will perform it reporting back the result to userspace.

But when a seccomp filter is loaded for the task issuing the syscall instead of directly looking up the syscall number in the architecture's syscall table the kernel will first call into seccomp and run the loaded seccomp filter.

Depending on whether a deny or allow approach is used for the seccomp filter any syscall that the filter is not handling specifically is either performed or denied reporting back a specified default value to the calling task.  If the requested syscall is supposed to be specifically handled by the seccomp filter the kernel can e.g. be caused to report back a specific error code.  This way, it is for example possible to have the kernel pretend like it doesn't know the `mount(2)` syscall by creating a seccomp filter that reports back `ENOSYS` whenever the task tries to call `mount(2)`.

But the way seccomp used to work isn't very dynamic.  Specifically, once a filter is loaded the decision whether or not the syscall is successful or not is fixed based on the policy expressed by the filter.  So there is no way to make a case-by-case decision which might come in handy in some scenarios.

In addition seccomp itself can't make a syscall actually succeed other than in the trivial way of reporting back success to the caller.  So seccomp will only allow the kernel to pretend that a syscall succeeded.  So while it is possible to instruct the kernel to return 0 for the `mount(2)` syscall it cannot actually be instructed to make the `mount(2)` syscall succeed.  So just making the seccomp filter return 0 for mounting a dedicated `ext4` disk device to `/mnt` will still not actually mount it at `/mnt`; it just pretends to the caller that it did.  Of course that is in itself already a useful property for a bunch of use-cases but it doesn't really help with the `mknod(2)` or `mount(2)` problem outlined above.

#### Extending Seccomp

So from the section above it should be clear that seccomp provides a few desirable properties that make it a natural candiate to look at to help solve our `mknod(2)` and `mount(2)` problem.  Since seccomp intercepts syscalls early in the syscall path it already gives us a hook into the syscall path of a given task.  What is missing though is a way to bring another task such as the LXD container manager into the picture.  Somehow we need to modify seccomp in a way that makes it possible for a container manager to not just be informed when a task inside the container performs a syscall it wants to be informed about but also how to make it possible to block the task until the container manager instructs the kernel to allow it to proceed.

The answer to these questions is the seccomp notifier.  This is as good a time as any to bring in some historical context.  The exact origins of the idea for a more dynamic way to intercept syscalls is probably not recoverable and it has been thrown around in unspecific form in various discussions but nothing serious every materialized.  The first concrete details around the seccomp notifier were conceived in early 2017 in the LXD team.  The first public talk around the basic idea for this feature was given by Stéphane Graber at the Linux Plumbers Conference 2017 during the Container's Microconference in Los Angeles.  The details of this talk are still listed [here](https://blog.linuxplumbersconf.org/2017/ocw/sessions/4795.html) here and I'm sure Stéphane can still provide the slides we came up with.  I didn't find a video recording even though I somehow thought we did have one.  If someone is really curious I can try to investigate with the Linux Plumbers committee.  After this talk implementation specifics were discussed in a hallway meeting later that day.  And after a long arduous journey the implementation was upstreamed by Tycho Andersen who used to be on the LXD team.  The rest is history^wchangelog.

#### Seccomp Notify - Syscall Interception 2.0

In its essence, the seccomp notify mechanism is simply a file descriptor (fd) for a specific seccomp filter.  When a container starts it will usually load a seccomp filter to restrict its attack surface.  That is even done for unprivileged containers even though it is not strictly necessary.

With the addition of the seccomp notifier a container wishing to have a subset of syscalls handled by another process can set the new `SECCOMP_RET_USER_NOTIF` flag on its seccomp filter.  This flag instructs the kernel to return a file descriptor to the calling task after having loaded its filter.  This file descriptor is a seccomp notify file descriptor.

Of course, the seccomp notify fd is not very useful to the task itself.  First, since it doesn't make a lot of sense apart from very weird use-cases for a task to listen for its own syscalls.  Second, because the task would likely block itself indefinitely pretty quickly without taking extreme care.

But what the task can do with the seccomp notifier is to hand to another task.  Usually the task that it will hand the seccomp notify fd to will be more privileged than itself.  For a container the most obvious candidate would be the container manager of course.

Since the seccomp notify fd is pollable it is possible to put it into an event loop such as `epoll(7)`, `poll(2)`, or `select(2)` and wait for the file descriptor to become readable, i.e. for the kernel to return `EPOLLIN` to userspace.  For the seccomp notify fd to become readable means that the seccomp filter it refers to has detected that one of the tasks it has been applied to has performed a syscall that is part of the policy it implements.  This is a complicated way of saying the kernel is notifying the container manager that a task in the container has performed a syscall it cares about, e.g. `mknod(2)` or `mount(2)`.

Put another way, this means the container manager can listen for syscall events for tasks running in the container.  Now instead of simply running the filter and immediately reporting back to the calling task the kernel will send a notification to the container manager on the seccomp notify fd and block the task performing the syscall.

After the seccomp notify fd indicates that it is readable the container manager can use the new `SECCOMP_IOCTL_NOTIF_RECV` `ioctl()` associated with seccomp notify fds to read a `struct seccomp_notif` message for the syscall.  Currently the data to be read from the seccomp notify fd includes the following pieces.  But please be aware that we are in the process of discussing potentially intrusive changes for future versions:

```c
struct seccomp_notif {
	__u64 id;
	__u32 pid;
	__u32 flags;
	struct seccomp_data data;
};
```

Let's look at this in a little more detail.  The `pid` field is the `pid` of the task that performed the syscall as seen in the caller's pid namespace.  To stay within the realm of our current examples, this is simply the pid of the task in the container the e.g. called `mknod(2)` as seen in the pid namespace of the container manager.  The `id` field is a unique identifier for the performed syscall.  This can be used to verify that the task is still alive and the syscall request still valid to avoid any race conditions caused by pid recycling.  The `flags` argument is currently unused and reserved for future extensions.

The `struct seccomp_data` argument is probably the most interesting one as it contains the really exciting bits and pieces:

```c
struct seccomp_data {
	int nr;
	__u32 arch;
	__u64 instruction_pointer;
	__u64 args[6];
};
```

The `int` field is the syscall number which can only be correctly interpreted relative to the `arch` field.  The `arch` field is the (audit) architecture for which this syscall was made.  This field is very relevant since compatible architectures (For the `x86` architectures this encompasses at least `x32`, `i386`, and `x86_64`. The `arm`, `mips`, and `power` architectures also have compatible "sub" architectures.) are stackable and the returned syscall number might be different than the current headers imply (For example, you could be making a syscall from a 32bit userspace on a 64bit kernel. If the intercepted syscall has different syscall numbers on 32 bit and on 64bit, for example syscall `foo()` might have syscall number 1 on 32 bit and 2 on 64 bit.  So the task reading the seccomp data can't simply assume that since it itself is running in a 32 bit environment the syscall number must be 1.  Rather, it must check what the audit `arch` is and then either check that the value of the syscall is 1 on 32 bit and 2 on 64 bit.  Otherwise the container manager might end up emulating `mount()` when it should be emulating `mknod()`.).  The `instruction_pointer` is set to the address of the instruction that performed the syscall. This is of course also architecture specific.  And last the `args` member are the syscall arguments that the task performed the syscall with.

The `args` need to be interpreted and treated differently depending on the syscall layout and their type.  If they are non-pointer arguments (`unsigned int` etc.) they can be copied into a local variable and interpreted right away.  But if they are pointer arguments they are offsets into the virtual memory of the task that performed the syscall.  In the latter case the memory needs to be read and copied before it can be interpreted.

Let's look at a concrete example to figure out why it is vital to know the syscall layout other than for knowing the types of the syscall arguments.  Say the performed syscall was `mount(2)`.  In order to interpret the `args` field correctly we look at the _syscall_ layout of `mount()`.  (Please note, that I'm stressing that we need to look at the layout of _syscall_ and the only reliable source for this is actually the kernel source code.  The Linux manpages often list the wrapper provided by the system's libc and these wrapper do not necessarily line-up with the syscall itself (compare the `waitid()` wrapper and the `waitid()` syscall or the various `clone()` syscall layouts).) From the layout of `mount(2)` we see that `args[0]` is a pointer argument identifying the source path, `args[1]` is another pointer argument identifying the target path, `args[2]` is a pointer argument identifying the filesystem type, `args[3]` is a non-pointer argument identifying the options, and `args[4]` is another pointer argument identifying additional mount options.

So if we were to be interested in the source path of this `mount(2)` syscall we would need to open the `/proc/<pid>/mem` file of the task that performed this syscall and e.g. use the `pread(2)` function with `args[0]` as the offset into the task's virtual memory and read it into a buffer at least the length of a standard path.  Alternatively, we can use a single syscall like `process_vm_readv(2)` to read multiple remote pointers at different locations all in one go. Once we have done this we can interpret it.

A friendly advice: in general it is a good idea for the container manager to read all syscall arguments _once_ into a local buffer and base its decisions on how to proceed on the data in this local buffer.  Not just because it will otherwise not be able for the container manager to interpret pointer arguments but it's also a possible attack vector since a sufficiently privileged attacker (e.g. a thread in the same thread-group) can write to `/proc/<pid>/mem` and change the contents of e.g. `args[0]` or any other syscall argument.  Also note, that the container manager should ensure that `/proc/<pid>` still refers to the same task after opening it by checking the validity of the syscall request via the `id` field and the associated `SECCOMP_IOCTL_NOTIF_ID_VALID` `ioctl()` to exclude the possibility of the task having exited, been reaped and its pid having been recycled.

But let's assume we have done all that.  Now that the container manager has the task's syscall arguments available in a local buffer it can interpret the syscall arguments.  While it is doing so the target task remains blocked waiting for the kernel to tell it to proceed.  After the container manager is done interpreting the arguments and has performed whatever action it wanted to perform it can use the `SECCOMP_IOCTL_NOTIF_SEND` `ioctl()` on the seccomp notify fd to tell the kernel what it should do with the blocked task's syscall.  The response is given in the form `struct seccomp_notif_resp`:

```c
struct seccomp_notif_resp {
	__u64 id;
	__s64 val;
	__s32 error;
	__u32 flags;
};
```

Let's look at this struct in a little more detail too.  The `id` field is set to the `id` of the syscall request to respond to and should correspond to the received `id` in the `struct seccomp_notif` that the container manager read via the `SECCOMP_IOCTL_NOTIF_RECV` `ioctl()` when the seccomp notify fd became readable.  The `val` field is the return value of the syscall and is only set if the `error` field is set to 0.  The `error` field is the error to return from the syscall and should be set to a negative `errno(3)` code if the syscall is supposed to fail (For example, to trick the caller into thinking that `mount(2)` is not supported on this kernel set `error` to `-ENOSYS`.).  The `flags` value can be used to tell the kernel to continue the syscall by setting the `SECCOMP_USER_NOTIF_FLAG_CONTINUE` flag which I added to be able to intercept `mount(2)` and other syscalls that are difficult for seccomp to filter efficiently because of the restrictions around pointer arguments.  More on that in a little bit.

With this machinery in place we are for now ;) done with the kernel bits.

#### Emulating Syscalls In Userspace

So what is the container manager supposed to do after having read and interpreted the syscall information for the task running in the container and telling the kernel to let the task continue.  Probably emulate it.  Otherwise we just have a fancy and less performant seccomp userspace policy (Please read my comments on why that is a _very_ bad idea.).

Emulating syscalls in userspace is not a very new thing to do.  It has been done for a long time.  For example, libc's can choose to emulate the `execveat(2)` syscall which allows a task to exec a program by providing a file descriptor to the binary instead of a path.  On a kernel that doesn't support the `execveat(2)` syscall the libc can emulate it by calling `exec(3)` with the path set to `/proc/self/fd/<nr>`.  The problem of course is that this emulation only works when the task in question actually uses the libc wrapper (`fexecve(3)` for our example).  Any task using `syscall(__NR_execveat, [...])` to perform the syscall without going through the provided wrapper will be bypassing libc and so libc doesn't know that the task wants to perform the `execveat(2)` syscall and will not be able to emulate it in case the kernel doesn't support it.

The seccomp notifier doesn't suffer from this problem since its syscall interception abilities aren't located in userspace at the library level but directly in the syscall path as we have seen.  This greatly expands the abilities to emulate syscalls.

So now we have all the kernel pieces in place to solve our `mknod(2)` and `mount(2)` problem in unprivileged containers.  Instead of simply letting the container fail on such harmless requests as creating the `/dev/zero` device node we can use the seccomp notifier to intercept the syscall and emulate it for the container in userspace by simply creating the device node for it.  Similarly, we can intercept `mount(2)` requests requiring the user to e.g. give us a list of allowed filesystems to mount for the container and performing the mount for the container.  We can even make this a lot safer by providing a user with the ability to specify a fuse binary that should be used when a task in the container tries to mount a filesystem.  We actually support this feature in LXD.  Since fuse is a safe way for unprivileged users to mount filesystems rewriting `mount(2)` requests is a great way to expose filesystems to containers.

In general, the possibilities of the seccomp notifier can't be overstated and we are extremely happy that this work is now not just fully integrated into the Linux kernel but also into both LXD and LXC.  As with many other technologies we have driven both in the upstream kernel and in userspace it directly benefits not just our users but all of userspace with the seccomp notifier seeing adoption in browsers and by other companies. A whole range of Travis workloads can now run in unprivileged LXD containers thanks to the seccomp notifier.

#### Seccomp Notify in action - LXD

After finishing the kernel bits we implemented support for it in LXD and the LXC shared library it uses.  Instead of simply exposing the raw seccomp notify fd for the container's seccomp filter directly to LXD each container connects to a multi-threaded socket that the LXD container manager exposes and on which it listens for new clients.  Clients here are new containers who the administrator has signed up for syscall supervisions through LXD.  Each container has a dedicated syscall supervisor which runs as a separate go routine and stays around for as long as the container is running.

When the container performs a syscall that the filter applies to a notification is generated on the seccomp notify fd.  The container then forwards this request including some additional data on the socket it connected to during startup by sending a unix message including necessary credentials.  LXD then interprets the message, checking the validity of the request, verifying the credentials, and processing the syscall arguments.  If LXD can prove that the request is valid according to the policy the administrator specified for the container LXD will proceed to emulate the syscall.  For `mknod(2)` it will create the device node for the container and for `mount(2)` it will mount the filesystem for the container.  Either by directly mounting it or by using a specified fuse binary for additional security.

If LXD manages to emulate the syscall successfully it will prepare a response that it will forward on the socket to the container.  The container then parses the message, verifying the credentials and will use the `SECCOMP_IOCTL_NOTIF_SEND` `ioctl()` sending a `struct seccomp_notif_resp` causing the kernel to unblock the task performing the syscall and reporting back that the syscall succeeded.  Conversely, if LXD fails to emulate the syscall for whatever reason or the syscall is not allowed by the policy the administrator specified it will prepare a message that instructs the container to report back that the syscall failed and unblocking the task.

#### Show Me!

Ok, enough talk.  Let's intercept some syscalls.  The following demo shows how LXD uses the seccomp notify fd to emulate the `mknod(2)` and `mount(2)` syscalls for an unprivileged container:

<script id="asciicast-285491" src="https://asciinema.org/a/285491.js" async></script>

#### Current Work and Future Directions

##### `SECCOMP_USER_NOTIF_FLAG_CONTINUE`

After the initial support for the seccomp notify fd landed we ran into limitations pretty quickly.  We realized we couldn't intercept the mount syscall.  Since the mount syscall has various pointer arguments it is difficult to write highly specific seccomp filters such that we only accept syscalls that we intended to intercept.  This is caused by seccomp not being able to handle pointer arguments.  They are opaque for seccomp.  So while it is possible to tell seccomp to only intercept `mount(2)` requests for real filesystems by only intercepting `mount(2)` syscalls where the `MS_BIND` flag is not set in the flags argument it is not possible to write a seccomp filter that only notifies the container manager about `mount(2)` syscalls for the `ext4` or `btrfs` filesystem because the filesystem argument is a pointer.

But this means we will inadvertently intercept syscalls that we didn't intend to intercept.  That is a generic problem but for some syscalls it's not really a big deal.  For example, we know that `mknod(2)` fails for all character and block devices in unprivileged containers.  So as long was we write a seccomp filter that intercepts only character and block device `mknod(2)` syscalls but no socket or fifo `mknod()` syscalls we don't have a problem.  For any character or block device that is not in the list of allowed devices in LXD we can simply instruct LXD to prepare a seccomp message that tells the kernel to report `EPERM` and since the syscalls would fail anyway there's no problem.

But _any_ system call that we intercepted as a consequence of seccomp not being able to filter on pointer arguments that would succeed in unprivileged containers would need to be emulated in userspace.  But this would of course include all `mount(2)` syscalls for filesystems that can be mounted in unprivileged containers.  I've listed a subset of them above.  It includes at least `tmpfs`, `proc`, `sysfs`, `devpts`, `cgroup`, `cgroup2` and probably a few others I'm forgetting.  That's not ideal.  We only want to emulate syscalls that we really have to emulate, i.e. those that would actually fail.

The solution to this problem was a patchset of mine that added the ability to continue an intercepted syscall.  To instruct the kernel to continue the syscall the `SECCOMP_USER_NOTIF_FLAG_CONTINUE` flag can be set in `struct seccomp_notif_resp`'s flag argument when instructing the kernel to unblock the task.

This is of course a very exciting feature and has a few readers probably thinking "Hm, I could implement a dynamic userspace seccomp policy." to which I want to very loudly respond "No, you can't!".  In general, the seccomp notify fd cannot be used to implement any kind of security policy in userspace.  I'm now going to mostly quote verbatim from my comment for the extension: The `SECCOMP_USER_NOTIF_FLAG_CONTINUE` flag must be used with extreme caution!  If set by the task supervising the syscalls of another task the syscall will continue.  This is problematic is inherent because of TOCTOU (Time of Check-Time of Use).  An attacker can exploit the time while the supervised task is waiting on a response from the supervising task to rewrite syscall arguments which are passed as pointers of the intercepted syscall.  It should be absolutely clear that this means that the seccomp notifier _cannot_ be used to implement a security policy on syscalls that read from dereferenced pointers in user space!  It should only ever be used in scenarios where a more privileged task supervises the syscalls of a lesser privileged task to get around kernel-enforced security restrictions when the privileged task deems this safe.  In other words, in order to continue a syscall the supervising task should be sure that another security mechanism or the kernel itself will sufficiently block syscalls if arguments are rewritten to something unsafe.

Similar precautions should be applied when stacking `SECCOMP_RET_USER_NOTIF` or `SECCOMP_RET_TRACE`.  For `SECCOMP_RET_USER_NOTIF` filters acting on the same syscall, the most recently added filter takes precedence.  This means that the new `SECCOMP_RET_USER_NOTIF` filter can override any `SECCOMP_IOCTL_NOTIF_SEND` from earlier filters, essentially allowing all such filtered syscalls to be executed by sending the response `SECCOMP_USER_NOTIF_FLAG_CONTINUE`.  Note that `SECCOMP_RET_TRACE` can equally be overriden by `SECCOMP_USER_NOTIF_FLAG_CONTINUE`.

##### Retrieving file descriptors `pidfd_getfd()`

Another extension that was added by [Sargun Dhillon](https://twitter.com/sargun) recently building on top of my pidfd work was to make it possible to retrieve file descriptors from another task.  This works even without the seccomp notifier since it is a new syscall but is of course especially useful in conjunction with it.

Often we would like to intercept syscalls such as `connect(2)`.  For example, the container manager might want to rewrite the `connect(2)` request to something other than the task intended for security reasons or because the task lacks the necessary information about the networking layout to connect to the right endpoint.  In these cases `pidfd_getfd(2)` can be used to retrieve a copy of the file descriptor of the task and perform the `connect(2)` for it.  This unblocks another wide range of use-cases.

For example, it can be used for further introspection into file descriptors than ss, or netstat would typically give you, as you can do things like run `getsockopt(2)` on the file descriptor, and you can use options like `TCP_INFO` to fetch a significant amount of information about the socket. Not only can you fetch information about the socket, but you can also set fields like `TCP_NODELAY`, to tune the socket without requiring the user's intervention. This mechanism, in conjunction can be used to build a rudimentary layer 4 load balancer where `connect(2)` calls are intercepted, and the destination is changed to a real server instead.

Early results indicate that this method can yield incredibly good latency as compared to other layer 4 load balancing techniques.

<div>
    <a href="https://plotly.com/~sargun/63/?share_key=TBxaZob2h9GiGD9LxVuFuE" target="_blank" title="Plot 63" style="display: block; text-align: center;"><img src="https://plotly.com/~sargun/63.png?share_key=TBxaZob2h9GiGD9LxVuFuE" alt="Plot 63" style="max-width: 100%;width: 600px;"  width="600" onerror="this.onerror=null;this.src='https://plotly.com/404.png';" /></a>
</div>



##### Injecting file descriptors `SECCOMP_NOTIFY_IOCTL_ADDFD`

Current work for the upcoming merge window is focussed on making it possible to inject file descriptors into a task.  As things stand, we are unable to intercept syscalls (Unless we share the file descriptor table with the task which is usually never the case for container managers and the containers they supervise.) such as `open(2)` that cause new file descriptors to be installed in the task performing the syscall.

The new seccomp extension effectively allows the container manager to instructs the target task to install a set of file descriptors into its own file descriptor table before instructing it to move on.  This way it is possible to intercept syscalls such as `open(2)` or `accept(2)`, and install (or replace, like `dup2(2)`) the container manager's resulting fd in the target task.

Christian
