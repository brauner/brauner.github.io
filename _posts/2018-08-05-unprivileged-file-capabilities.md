---
layout: post
title: 'Unprivileged File Capabilities'
---

![alt text](https://linuxcontainers.org/static/img/containers.png)

#### Introduction

File capabilities (fcaps) are capabilities associated with - well - files,
usually a binary. They can be used to temporarily elevate privileges for
unprivileged users in order to accomplish a privileged task. This allows
various tools to drop the dangerous setuid (suid) or setgid (sgid) bits in
favor of fcaps.

While fcaps are supported since Linux `2.6.24` they could only be set in the
initial user namespace. If they would have been allowed to be set by root in
a non-initial user namespace then any unprivileged user on the host would have
been able to map their own uid to root in a new user namespace, set fcaps that
would grant more privileges to them, and then execute the binary with elevated
privileged on the host. This also means that until recently it was not safe to
use fcaps in unprivileged containers, i.e. containers using user namespaces.
The good news is that starting with Linux kernel version `4.14` it is possible
to set **fcaps in user namespaces**.

#### Kernel Patchset

The patchset to enable this has been contributed by Serge Hallyn,
a co-maintainer and core developer of the [LXD](https://github.com/lxc/lxd) and
[LXC](https://github.com/lxc/lxc) projects:

```
commit 8db6c34f1dbc8e06aa016a9b829b06902c3e1340
Author: Serge E. Hallyn <serge@hallyn.com>
Date:   Mon May 8 13:11:56 2017 -0500

    Introduce v3 namespaced file capabilities
```

#### LXD Now Preserves File Capabilities In User Namespaces

In parallel to the kernel patchset we have now enabled
[LXD](https://github.com/lxc/lxd) to preserve fcaps in user namespaces. This
means if your kernel supports namespaced fcaps
[LXD](https://github.com/lxc/lxd) will preserve them whenever unprivileged
containers are created, or when their idmapping is changed. No matter if you go
from privileged to unpriviliged or the other way around. Your filesystem
capabilities will be right there with you. In other news, there is now little
to no use for the `suid` and `sgid` bits even in unprivileged containers.

This is something that the [Linux Containers Project](https://linuxcontainers.org)
has wanted for a long time and we are happy that we are the first runtime to
fully support this feature.

If all of the above either makes no sense to you or you're asking yourself what
is so great about this because some distros have been using fcaps for a long
time don't worry we'll try to shed some light on all of this.

#### The dark ages: `suid` and `sgid` binaries

Not too long ago the suid and sgid bits were the only well-known mechanism to
temporarily grant elevated privileges to unprivileged users executing a binary.
Once some or all of the following binaries where `suid` or `sgid` binaries on
most distros:

- `ping`
- `newgidmap`
- `newuidmap`
- `mtr-packet`

The binary that most developers will have already used is the `ping` binary.
It's convenient to just check whether a connection to the internet has been
established successfully by pinging a random website. It's such a common tool
that most people don't even think about it needing any sort of privilege. In
fact it does require privileges. `ping` wants to open sockets of type
`SOCK_RAW` but the kernel prevents unprivileged users from using sockets of
type `SOCK_RAW` because it would allow them to e.g. send `ICMP` packages
directly. But `ping` seems like a binary that is useful to unprivileged users
as well as safe. Short of a better mechanism the most obvious choice is to have
it be owned by uid `0` and set the `suid` bit.

```shell
chb@conventiont|~
> perms /bin/ping
-rwsr-xr-x 4755 /bin/ping
```

You can see the little `s` in the permissions. This indicates that this version
of `ping` has the `suid` bit set. Hence, if called it will run as uid `0`
independent of the uid of the caller. In short, if my user has uid `1000` and
calls the `ping` binary `ping` will still run with uid `0`.

While the `suid` mechanism gets the job done it is also wildly inappropriate.
`ping` does need elevated privileges in one specific area. But by setting the
`suid` bit and having `ping` be owned by uid `0` we're granting it all kinds of
privileges, in fact all privileges. If there ever is a major security sensitive
bug in a `suid` binary it is trivial for anyone to exploit the fact that it
runs as uid `0`.

Of course, the kernel has all kinds of security mechanisms to deflate the
impact of the `suid` and `sgid` bits. If you `strace` an `suid` binary the
`suid` bit will be stripped, there are complex rules regarding `execve()`ing
a binary that has the `suid` bit set, and the `suid` bit is also dropped when
the owner of the binary in question changes, i.e. when you call `chown()` on
it. Still these are all migitations for something that is inherently dangerous
because it grants too much for too little gain. It's like someone asking for
a little sugar and you handing out the key to your house. To [quote
Eric](https://lwn.net/Articles/420993/):

> Frankly being able to raise the priveleges of an existing process is such
a dangerous mechanism and so limiting on system design that I wish someone
would care, and remove all suid, sgid, and capabilities use from a distro. It
is hard to count how many neat new features have been shelved because of the
requirement to support suid root executables.

#### Capabilities and File Capabilities

This is where capabilities come into play [^3]. Capabilities start from the
idea that the root privilege could as well be split into subsets of privileges.
Whenever something requests to perform an operation that requires privileges it
doesn't have we can grant it a very specific subset instead of all privileges
at once [^1]. For example, the `ping` binary would only need the `CAP_NET_RAW`
capability because it is the capability that regulates whether a process can
open `SOCK_RAW` sockets.

Capabilities are associated with processes and files. Granted, Linux
capabilities are not the cleanest or easiest concept to grasp. But I'll try to
shed some light. In essence, capabilities can be present in four different
types of sets. The kernel performs checks against a process by looking at its
**effective** capability set, i.e. the capabilities the process has at the time
of trying to perform the operation. The rest of the capability sets
are (glossing over details now for the sake of brevity) basically used for
calculating each other including the effective capability set. There are
**permitted** capabilities, i.e. the capabilities a process is allowed to raise
in the **effective** set, **inheritable** capabilities, i.e. capabilities that
should be (but are only under certain restricted conditions) preserved across
an `execve()`, and **ambient** capabilities that are there to fix the
shortcomings of **inheritable** capabilities, i.e. they are there to allow
unprivileged processes to preserve capabilities across an `execve()` call. [^2]
Last but not least we have file capabilities, i.e. capabilities that are
attached to a file. When such a file is `execve()`ed the associated fcaps are
taken into account when calculating the permissions after the `execve()`.

#### Extended attributes and File Capabilities

The part most users are confused about is how capabilities get associated with
files. This is where extended attributes (xattr) come into play. xattrs are
`<key>:<value>` pairs that can be associated with files. They are stored
on-disk as part of the metadata of a file.
The `<key>` of an xattr will always be a string identifying the attribute in
question whereas the `<value>` can be arbitrary data, i.e. it can be another
string or binary data.
Note that it is not guaranteed nor required by the kernel that a filesystem
supports xattrs. While the virtual filesystem (vfs) will handle all core
permission checks, i.e. it will verify that the caller is allowed to set the
requested xattr but the actual operation of writing out the xattr on disk will
be left to the filesystem. Without going into the specifics the callchain
currently is:
```c
SYSCALL_DEFINE5(setxattr, const char __user *, pathname,
                const char __user *, name, const void __user *, value,
                size_t, size, int, flags)
|
-> static int path_setxattr(const char __user *pathname,
                            const char __user *name, const void __user *value,
                            size_t size, int flags, unsigned int lookup_flags)
   |
   -> static long setxattr(struct dentry *d, const char __user *name,
                           const void __user *value, size_t size, int flags)
      |
      -> int vfs_setxattr(struct dentry *dentry, const char *name,
                          const void *value, size_t size, int flags)
         |
         -> int __vfs_setxattr_noperm(struct dentry *dentry, const char *name,
                                      const void *value, size_t size, int flags)
```

and finally `__vfs_setxattr_noperm()` will call

```c
int __vfs_setxattr(struct dentry *dentry, struct inode *inode, const char *name,
                   const void *value, size_t size, int flags)
{
        const struct xattr_handler *handler;

        handler = xattr_resolve_name(inode, &name);
        if (IS_ERR(handler))
                return PTR_ERR(handler);
        if (!handler->set)
                return -EOPNOTSUPP;
        if (size == 0)
                value = "";  /* empty EA, do not remove */
        return handler->set(handler, dentry, inode, name, value, size, flags);
}
```

The `__vfs_setxattr()` function will then call `xattr_resolve_name()` which
will find and return the appropriate handler for the xattr in the list `struct
xattr_handler` of the corresponding filesystem. If the filesystem has a handler
for the xattr in question it will return it and the attribute will be set and
if not `EOPNOTSUPP` will be surfaced to the caller.

For this article we will only focus on the permission checks that the vfs
performs not on the filesystem specifics. An important thing to note is that
different xattrs are subject to different permission checks by the vfs. First,
the vfs regulates what types of xattrs are supported in the first place. If you
look at the `xattr.h` header you will find all supported xattr namespaces. An
xattr namespace is essentially nothing but a prefix like `security.`. Let's
look at a few examples from the `xattr.h` header:

```c
#define XATTR_SECURITY_PREFIX "security."
#define XATTR_SECURITY_PREFIX_LEN (sizeof(XATTR_SECURITY_PREFIX) - 1)

#define XATTR_SYSTEM_PREFIX "system."
#define XATTR_SYSTEM_PREFIX_LEN (sizeof(XATTR_SYSTEM_PREFIX) - 1)

#define XATTR_TRUSTED_PREFIX "trusted."
#define XATTR_TRUSTED_PREFIX_LEN (sizeof(XATTR_TRUSTED_PREFIX) - 1)

#define XATTR_USER_PREFIX "user."
#define XATTR_USER_PREFIX_LEN (sizeof(XATTR_USER_PREFIX) - 1)
```

Based on the detected prefix the vfs will decide what permission checks to
perform. For example, the `user.` namespace is not subject to very strict
permission checks since it exists to allow users to store arbitrary
information.
However, some xattrs are subject to very strict permission checks since they
allow to change privileges. For example, this affects the `security.`
namespace. In fact, the `xattr.h` header even exposes a specific `capability`
suffix to use with the `security.` namespace:

```c
#define XATTR_CAPS_SUFFIX "capability"
#define XATTR_NAME_CAPS XATTR_SECURITY_PREFIX XATTR_CAPS_SUFFIX
```

As you might have figured out file capabilities are associated with the
`security.capability` xattr.

In contrast to other xattrs the value associated with the `security.capability`
xattr key is not a string but binary data. The actual implementation is a `C`
`struct` that contains bitmasks of capability flags. To actually set file
capabilities userspace would usually use the `libcap` library because the
low-level bits of the implementation are not very easy to use. Let's say a user
wanted to associate the `CAP_NET_RAW` capability with the `ping` binary on
a system that only supports non-namespaced file capabilities. Then this is the
minimum that you would need to do in order to set `CAP_NET_RAW` in the
effective and permitted set of the file:

```c
/*
 * Do not simply copy this code. For the sake of brevity I e.g. omitted
 * handling the necessary endianess translation. (Not to speak of the apparent
 * ugliness and missing documentation of my sloppy macros.)
 */

struct vfs_cap_data xattr = {0};

#define raise_cap_permitted(x, cap_data)   cap_data.data[(x)>>5].permitted   |= (1<<((x)&31))
#define raise_cap_inheritable(x, cap_data) cap_data.data[(x)>>5].inheritable |= (1<<((x)&31))

raise_cap_permitted(CAP_NET_RAW, xattr);
xattr.magic_etc = VFS_CAP_REVISION_2 | VFS_CAP_FLAGS_EFFECTIVE;

setxattr("/bin/ping", "security.capability", &xattr, sizeof(xattr), 0);
```

After having done this we can look at the ping binary and use the `getcap`
binary to check whether we successfully set the `CAP_NET_RAW` capability on the
`ping` binary. Here's a little demo:

[![asciicast](https://asciinema.org/a/195195.png)](https://asciinema.org/a/195195)

#### Setting Unprivileged File Capabilities

On kernels that support namespaced file capabilities the straightforward way to
set a file capability is to attach to the user namespace in question as root
and then simply perform the above operations. The kernel will then
transparently handle the translation between a non-namespaced and a namespaced
capability by recording the rootid from the kernel's perspective (the `kuid`).

However, it is also possible to set file capabilities in lieu of another user
namespace. In order to do this the code above needs to be changed slightly:

```c
/* 
 * Do not simply copy this code. For the sake of brevity I e.g. omitted
 * handling the necessary endianess translation. (Not to speak of the apparent
 * ugliness and missing documentation of my sloppy macros.)
 */

struct vfs_ns_cap_data ns_xattr = {0};

#define raise_cap_permitted(x, cap_data)   cap_data.data[(x)>>5].permitted   |= (1<<((x)&31))
#define raise_cap_inheritable(x, cap_data) cap_data.data[(x)>>5].inheritable |= (1<<((x)&31))

raise_cap_permitted(CAP_NET_RAW, ns_xattr);
ns_xattr.magic_etc = VFS_CAP_REVISION_3 | VFS_CAP_FLAGS_EFFECTIVE;
ns_xattr.rootid = 1000000;

setxattr("/bin/ping", "security.capability", &ns_xattr, sizeof(ns_xattr), 0);
```

As you can see the `struct` we use has changed. Instead of using
`struct vfs_cap_data` we are now using `struct vfs_ns_cap_data` which has
gained an additional field `rootid`. In our example we are setting the `rootid`
to `1000000` which in my example is the `rootid` of uid `0` in the container's
user namespace **as seen from the host**. Additionally, we set the `magic_etc`
bit for the fcap version that the vfs is expected to support to
`VFS_CAP_REVISION_3`.

[![asciicast](https://asciinema.org/a/195198.png)](https://asciinema.org/a/195198)

As you can see from the asciicast we can't execute the `ping` binary as an
unprivileged user on the host since the fcaps is namespaced and associated with
uid `1000000`. But if we copy that binary to a container where this uid is
mapped to uid `0` we can now call `ping` as an unprivileged user.

So let's look at an actual unprivileged container and let's set the
`CAP_NET_RAW` capability on the `ping` binary in there:

[![asciicast](https://asciinema.org/a/195209.png)](https://asciinema.org/a/195209)

#### Some Implementation Details

As you have seen above a new `struct vfs_ns_cap_data` has been added to the
kernel:
```c
/*
 * same as vfs_cap_data but with a rootid at the end
 */
struct vfs_ns_cap_data {
        __le32 magic_etc;
        struct {
                __le32 permitted;    /* Little endian */
                __le32 inheritable;  /* Little endian */
        } data[VFS_CAP_U32];
        __le32 rootid;
};
```

In the end this `struct` is what the kernel expects to be passed and which it
will use to calculate fcaps. The location of the permitted and inheritable set
in `struct vfs_ns_cap_data` are obvious but the effective set seems to be
missing. Whether or not effective caps are set on the file is determined by
raising the `VFS_CAP_FLAGS_EFFECTIVE` bit in the `magic_etc` mask. The
`magic_etc` member is also used to tell the kernel which fcaps version the vfs
is expected to support. The kernel will verify that either `XATTR_CAPS_SZ_2` or
`XATTR_CAPS_SZ_3` are passed as size and are correctly paired with
the `VFS_CAP_REVISION_2` and `VFS_CAP_REVISION_3` flag. If `XATTR_CAPS_SZ_2` is
set then the kernel will not try to look for a `rootid` field in the `struct`
it received, i.e. even if you pass a `struct vfs_ns_cap_data` with a `rootid`
but set `XATTR_CAPS_SZ_2` as size parameter and `VFS_CAP_REVISION_2` in
`magic_etc` the kernel will be able to ignore the `rootid` field and instead
use the `rootid` of the current user namespace. This allows the kernel to
transparently translate from `VFS_CAP_REVISION_2` to `VFS_CAP_REVISION_3`
fcaps. The main translation mechanism can be found in `cap_convert_nscap()` and
`rootid_from_xattr()`:

```c
/*
* User requested a write of security.capability.  If needed, update the
* xattr to change from v2 to v3, or to fixup the v3 rootid.
*
* If all is ok, we return the new size, on error return < 0.
*/
int cap_convert_nscap(struct dentry *dentry, void **ivalue, size_t size)
{
        struct vfs_ns_cap_data *nscap;
        uid_t nsrootid;
        const struct vfs_cap_data *cap = *ivalue;
        __u32 magic, nsmagic;
        struct inode *inode = d_backing_inode(dentry);
        struct user_namespace *task_ns = current_user_ns(),
                *fs_ns = inode->i_sb->s_user_ns;
        kuid_t rootid;
        size_t newsize;

        if (!*ivalue)
                return -EINVAL;
        if (!validheader(size, cap))
                return -EINVAL;
        if (!capable_wrt_inode_uidgid(inode, CAP_SETFCAP))
                return -EPERM;
        if (size == XATTR_CAPS_SZ_2)
                if (ns_capable(inode->i_sb->s_user_ns, CAP_SETFCAP))
                        /* user is privileged, just write the v2 */
                        return size;

        rootid = rootid_from_xattr(*ivalue, size, task_ns);
        if (!uid_valid(rootid))
                return -EINVAL;

        nsrootid = from_kuid(fs_ns, rootid);
        if (nsrootid == -1)
                return -EINVAL;

        newsize = sizeof(struct vfs_ns_cap_data);
        nscap = kmalloc(newsize, GFP_ATOMIC);
        if (!nscap)
                return -ENOMEM;
        nscap->rootid = cpu_to_le32(nsrootid);
        nsmagic = VFS_CAP_REVISION_3;
        magic = le32_to_cpu(cap->magic_etc);
        if (magic & VFS_CAP_FLAGS_EFFECTIVE)
                nsmagic |= VFS_CAP_FLAGS_EFFECTIVE;
        nscap->magic_etc = cpu_to_le32(nsmagic);
        memcpy(&nscap->data, &cap->data, sizeof(__le32) * 2 * VFS_CAP_U32);

        kvfree(*ivalue);
        *ivalue = nscap;
        return newsize;
}
```

#### Conclusion

Having fcaps available in user namespaces just makes the argument to always use
unprivileged containers even stronger.
The [Linux Containers Project](https://linuxcontainers.org) is also working on
a bunch of other kernel- and userspace features to improve unprivileged
containers even more. Stay tuned! :)

Christian

[^1]: Exactly how to split up the root privilege and how exactly privileges should be implemented (e.g. should they be attached to file descriptors, should they be attached to inodes, etc.) is a good argument to have. For the sake of this article we will skip this discussion and assume the Linux implementation of POSIX capabilities.

[^2]: If people are super keen and request this I can make a longer post how exactly they all relate to each other and possibly look at some of the implementation details too.

[^3]: While capabilities provide a **better** mechanism to temporarily and **selectively** grant privileges to unprivileged processes they are by no means inherently safe. Setting fcaps should still be done rarely. If privilege escalation happens via `suid` or `sgid` bits or fcaps doesn't matter in the end: it's still a privilege escalation.
