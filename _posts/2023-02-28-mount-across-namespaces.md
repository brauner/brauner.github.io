---
layout: post
title: Mounting into mount namespaces
---

# Introduction

Early on when the `LXD` project was started we were clear that we wanted to make it possible to change settings while the container is running.
On of the very first things that came to our mind was making it possible to insert new mounts into a running container.
When I was still at Canonical working on `LXD` we quickly realized that inserting mounts into a running container would require a lot of creativity given the limitations of the api.

Back then the only way to create mounts or change mount option was by using the `mount(2)` system call.
The mount system call multiplexes a lot of different operations.
For example, it doesn't just allow the creation of new filesystem mounts but also handles bind mounts and mount option changes.
Mounting is overall a pretty complex operation as it doesn't just involve path lookup but also needs to handle mount propagation and filesystem specific and generic mount options.

I want to take a look at our legacy solution to this problem and a new approach that I've used and that has existed for a while but never talked about widely.

# Creative uses of `mount(2)`

Before `openat2(2)` came along adding mounts to a container during startup was difficult because there was always the danger of symlink attacks.
A mount source or target path could be specified containing symlinks that would allow processes in the container to escape to the host filesystem.
These attacks used to be quite common and there was no straightforward solution available; at least not before the `RESOLVE_*` flag namespace of `openat2(2)` improved things so considerably that symlink attacks on new kernels can be effectively blocked.

But before `openat2()` symlink attacks when mounting could only be prevented with very careful coding and a rather elaborate algorithm.
I won't go into too much detail but it is roughly done by verifying each path component in userspace using `O_PATH` file descriptors making sure that the paths point into the container's rootfs.

But even if you verified that the path is sane and you hold a file descriptor to the last component you still need to solve the problem that `mount(2)` only operates on paths.
So you are still susceptible to symlink attacks as soon as you call `mount(source, target, ...)`.

The way we solved this problem was by realizing that `mount(2)` was perfectly happy to operate on `/proc/self/fd/<nr>` paths
(This is similar to how `fexecve()` used to work before the addition of the `execveat()` system call.).
So we could verify the whole path and then open the last component of the source and target paths at which point we could call `mount("/proc/self/fd/1234", "/proc/self/fd/5678", ...)`.

We immediately thought that if `mount(2)` allows you to do that then we could easily use this to mount into namespaces.
So if the container is running it its mount namespace we could just create a bind mount on the host, open the newly created bind mount and then change to the container's mount namespace (and it's owning user namespace) and then simply call `mount("/proc/self/fd/1234", "/mnt", ...)`.
In pseudo C code it would look roughly:

```c
fd_mnt = openat(-EBADF, "/opt", O_PATH, ...);
setns(fd_userns, CLONE_NEWUSER);
setns(fd_mntns, CLONE_NEWNS);
mount("/proc/self/fd/fd_mnt", "/mnt", ...);
```

However, this isn't possible as the kernel will enforce that the mounts that the source and target paths refer to are located in the caller's mount namespace.
Since the caller will be located in the container's mount namespace after the `setns()` call but the source file descriptors refers to a mount located in the host's mount namespace this check fails.
The semantics behind this are somewhat sane and straightforward to understand so there was no need to change them even though we were tempted.
Back then it would've also meant that adding mounts to containers would've only worked on newer kernels and we were quite eager to enable this feature for kernels that were already released.

# Mount namespace tunnels

So we came up with the idea of mount namespace tunnels.
Since we spearheaded this idea it has been picked up by various projects such as `systemd` for system services and it's own `systemd-nspawn` container runtime.

The general idea as based on the observation that mount propagation can be used to function like a tunnel between mount namespaces:

```
mount --bind /opt /opt
mount --make-private /opt
mount --make-shared /opt
# Create new mount namespace with all mounts turned into dependent mounts.
unshare --mount --propagation=slave
```

and then create a mount on or beneath the shared `/opt` mount on the host:

```
mkdir /opt/a
mount --bind /tmp /opt/a
```

then the new mount of `/tmp` on the dentry `/opt/a` will propagate into the mount namespace we created earlier.
Since the `/opt` mount at the `/opt` dentry in the new mount namespace is a dependent mount we can now move the mount to its final location:

```
mount --move /opt/a /mnt
```

As a last step we can unmount `/opt/a` in the host mount namespace.
And as long as the `/mnt` dentry doesn't reside on a mount that is a dependent mount of `/opt`'s peer group the unmount of `/opt/a` we just performed on the host will only unmount the mount in the host mount namespace.

There are various problems with this solution:

* It's complex.
* The container manager needs to set up the mount tunnel when the container starts.
  In other words, it needs to part of the architecture of the container which is always unfortunate.
* The mount at the endpoint of the tunnel in the container needs to be protected from being unmounted.
  Otherwise the container payload can just unmount the mount at its end of the mount tunnel and prevent the insertion of new mounts into the container.

# Mounting into mount namespaces

A few years ago a new mount api made it into the kernel.
Shortly after I've also added the `mount_setattr(2)` system call.
Since then I've been expanding the abilities of this api and to put it to its full use.

Unfortunately the adoption of the new mount api has been slow.
Mostly, because people don't know about it or because they don't yet see the many advantages it offers over the old one.
But with the next release of the `mount(8)` binary a lot of us use the new mount api will be used whenever possible.

I won't be covering all the features that the mount api offers.
This post just illustrates how the new mount api makes it possible to mount into mount namespaces and let's us get rid of the complex mount propagation scheme.

Luckily, the new mount api is designed around file descriptors.

## Filesystem Mounts

To create a new filesystem mount using the old mount api is simple:

```
mount("/dev/sda", "/mnt", "xfs", ...);
```

We pass the source, target, and filesystem type and potentially additional mount options.
This single system call does a lot behind the scenes.
A new superblock will be allocated for the filesystem, mount options will be set, a new mount will be created and attached to a mountpoint in the caller's mount namespace.

In the new mount api the various steps are split into separate system calls.
While this makes mounting more complex it allows allows for greater flexibility.
Mounting doesn't have to be a fast operation and never has been.

So in the new mount api we would create a new filesystem mount with the following steps:

```c
/* Create a new filesystem context. */
fd_fs = fsopen("xfs");

/*
 * Set the source of the filsystem mount. Whether or not this is required
 * depends on the type of filesystem of course. For example, mounting a tmpfs
 * filesystem would not require us to set the "source" property as it's not
 * backed by a block device. 
 */
fsconfig(fd_fs, FSCONFIG_SET_STRING, "source", "/dev/sda", 0);

/* Actually create the superblock and prepare to allocate a mount. */
fsconfig(fd_fs, FSCONFIG_CMD_CREATE, NULL, NULL, 0);
```

The `fd_fs` file descriptor refers to VFS context object that doesn't concern us here.
Let it suffice that it is an opaque object that can only be used to configure
the superblock and the filesystem until `fsmount()` is called:

```c
/* Create a new detached mount and return an O_PATH file descriptor refering to the mount. */
fd_mnt = fsmount(fd_fs, 0, 0);
```

The `fsmount()` call will turn the context file descriptor into an `O_PATH` file descriptor that refers to a detached mount.
A detached mount is a mount that isn't attached to any mount namespace.

## Bind Mounts

The old mount api created bind mounts via:

```c
mount("/opt", "/mnt", MNT_BIND, ...)
```

and recursive bind mounts via:

```c
mount("/opt", "/mnt", MNT_BIND | MS_REC, ...)
```

Most people however will be more familiar with `mount(8)`:

```
mount --bind /opt /mnt
mount --rbind / /mnt
```

Bind mounts play a major role in container runtimes and system services as run by `systemd`.

The new mount api supports bind mounts through the `open_tree()` system call.
Calling `open_tree()` on an existing mount will just return an `O_PATH` file descriptor referring to that mount.
But if `OPEN_TREE_CLONE` is specified `open_tree()` will create a detached mount and return an `O_PATH` file descriptor.
That file descriptor is indistinguishable from an `O_PATH` file descriptor returned from the earlier `fsmount()` example:

```c
fd_mnt = open_tree(-EBADF, "/opt", OPEN_TREE_CLONE, ...)
```

creates a new detached mount of `/opt` and:

```c
fd_mnt = open_tree(-EBADF, "/", OPEN_TREE_CLONE | AT_RECURSIVE, ...)
```

would create a new detached copy of the whole rootfs mount tree.

### Attaching detached mounts

As mentioned before the file descriptor returned from `fsmount()` and `open_tree(OPEN_TREE_CLONE)` refers to a detached mount in both cases.
The mount it refers to doesn't appear anywhere in the filesystem hierarchy.
Consequently, the mount can't be found by lookup operations going through the filesystem hierarchy.
The new mount api thus provides an elegant mechanism for:

```c
mount("/opt", "/mnt", MS_BIND, ...);
fd_mnt = openat(-EABDF, "/mnt", O_PATH | O_DIRECTORY | O_CLOEXEC, ...);
umount2("/mnt", MNT_DETACH);
```

and with the added benefit that the mount never actually had to appear anywhere in the filesystem hierarchy and thus never had to belong to any mount namespace.
This alone is already a very powerful tool but we won't go into depth today.

Most of the time a detached mount isn't wanted however.
Usually we want to make the mount visible in the filesystem hierarchy so other user or programs can access it.
So we need to attach them to the filesystem hierarchy.

In order to attach a mount we can use the `move_mount()` system call.
For example, to attach the detached mount `fd_mnt` we create before we can use:

```c
move_mount(fd_mnt, "", -EBADF, "/mnt", MOVE_MOUNT_F_EMPTY_PATH);
```

This will attach the detached mount of `/opt` at the `/mnt` dentry on the `/` mount.
What this means is that the `/opt` mount will be inserted into the mount namespace that the caller is located in at the time of calling `move_mount()`.
(The kernel has very tight semantics here. For example, it will enforce that the caller has `CAP_SYS_ADMIN` in the owning user namespace of its mount namespace.
It will also enforce that the mount the `/mnt` dentry is located on belongs to the same mount namespace as the caller.)

After `move_mount()` returns the mount is permanently attached.
Even if it is unmounted while still pinned by a file descriptor will it still belong to the mount namespace it was attached to.
In other words, `move_mount()` is an irreversible operation.

The main point is that before `move_mount()` is called a detached mount doesn't belong to any mount namespace and can thus be freely moved around.

## Mounting a new filesystem into a mount namespace

To mount a filesystem into a new mount namespace we can make use of the split between configuring a filesystem context and creating a new superblock and actually attaching the mount to the filesystem hiearchy:

```c
```
fd_fs = fsopen("xfs");
fsconfig(fd_fs, FSCONFIG_SET_STRING, "source", "/dev/sda", 0);
fsconfig(fd_fs, FSCONFIG_CMD_CREATE, NULL, NULL, 0);
fd_mnt = fsmount(fd_fs, 0, 0);
```

For filesystems that require host privileges such as `xfs`, `ext4`, or `btrfs` (and many others) these steps can be performed by a privileged container or pod manager with sufficient privileges.
However, once we have created a detached mounts we are free to attach to whatever mount and mountpoint we have privilege over in the target mount namespace.
So we can simply attach to the user namespace and mount namespace of the container:

```c
setns(fd_userns);
setns(fd_mntns);
```

and then use

```c
move_mount(fd_mnt, "", -EBADF, "/mnt", MOVE_MOUNT_F_EMPTY_PATH);
```

to attach the detached mount anywhere we like in the container.

## Mounting a new bind mount into a mount namespace

A bind mount is even simpler.
If we want to share a specific host directory with the container we can just have the container manager call:

```c
fd_mnt = open_tree(-EBADF, "/opt", OPEN_TREE_CLOEXEC | OPEN_TREE_CLONE);
```

to allocate a new detached copy of the mount and then attach to the user and mount namespace of the container:

```c
setns(fd_userns);
setns(fd_mntns);
```

and as above we are free to attach the detached mount anywhere we like in the container.

# Conclusion

This is really it.
It's as simple as it sounds.
This is a powerful delegation mechanism that we've been making heavy use of in `LXD` and is the proper way how to insert mounts into mount namespaces on newer kernels.

# Contributing fixes, improvements, or corrections to this post

If you have fixes, improvements, or corrections feel free to email them to me or simply open a pull request against [the repository for this blog](https://github.com/brauner/brauner.github.io).
