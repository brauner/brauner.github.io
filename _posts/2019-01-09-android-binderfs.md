---
layout: post
title: 'Android Binderfs'
---

[![asciicast](https://asciinema.org/a/220715.svg)](https://asciinema.org/a/220715)

#### Introduction

[Android Binder][3] is an inter-process communication (IPC) mechanism. It is
heavily used in all [Android devices][2]. The binder kernel driver has been
present in the [upstream Linux kernel][1] for quite a while now.

Binder has been a controversial patchset (see this [lwn][6] article as an
example). Its design was considered wrong and to violate certain core kernel
design principles (e.g. a task should never touch another tasks file descriptor
table). Most kernel developers were not a fan of binder.

Recently, the upstream binder code has fortunately been reworked significantly
(e.g. it does not touch another tasks file descriptor table anymore, the
locking is very fine-grained now, etc.).

With Android being one of the major operating systems (OS) for a vast number of
devices there is simply no way around binder.

#### The Android Service Manager

The binder IPC mechanism is accessible from userspace through device nodes
located at `/dev`. A modern Android system will allocate three device nodes:

- `/dev/binder`
- `/dev/hwbinder`
- `/dev/vndbinder`

serving different purposes. However, the logic is the same for all three of
them. A process can call `open(2)` on those device nodes to receive an fd which
it can then use to issue requests via `ioctl(2)`s.
Android has a service manager which is used to translate addresses to bus
names and only the address of the service manager itself is well-known. The
service manager is registered through an `ioctl(2)` and there can only be
a single service manager. This means once a service manager has grabbed hold of
binder devices they cannot be (easily) reused by a second service manager.

#### Running Android in Containers

This matters as soon as multiple instances of Android are supposed to be run.
Since they will all need their own private binder devices. This is a use-case
that arises pretty naturally when running Android in system containers. People
have been doing this for a long time with [LXC][5]. A project that has set out
to make running Android in [LXC][5] containers very easy is [Anbox][4].
[Anbox][4] makes it possible to run hundreds of Android containers.

To properly run Android in a container it is necessary that each container has
a set of private binder devices.

#### Statically Allocating binder Devices

Binder devices are currently statically allocated at *compile time*. Before
compiling a kernel the `CONFIG_ANDROID_BINDER_DEVICES` option needs to bet set
in the kernel config (Kconfig) containing the names of the binder devices to
allocate at boot. By default it is set as:
```
CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder"
```
To allocate additional binder devices the user needs to specify them with this
Kconfig option. This is problematic since users need to know how many
containers they will run at maximum and then to calculate the number of devices
they need so they can specify them in the Kconfig. When the maximum number of
needed binder devices changes after kernel compilation the only way to get
additional devices is to recompile the kernel.

#### Problem 1: Using the misc major Device Number

This situation is aggravated by the fact that binder devices use the misc major
number in the kernel. Each device node in the Linux kernel is identified by
a major and minor number. A device can request its own major number. If it does
it will have an exclusive range of minor numbers it doesn't share with anything
else and is free to hand out. Or it can use the misc major number. The misc
major number is shared amongst different devices. However, that also means the
number of minor devices that can be handed out is limited by all users of misc
major. So if a user requests a very large number of binder devices in their
Kconfig they might make it impossible for anyone else to allocate minor
numbers. Or there simply might not be enough to allocate for itself.

#### Problem 2: Containers and IPC namespaces

All of those binder devices requested in the Kconfig via
`CONFIG_ANDROID_BINDER_DEVICES` will be allocated at boot and be placed in the
hosts `devtmpfs` mount usually located at `/dev` or - depending on the
`udev(7)` implementation - will be created via `mknod(2)` - by `udev(7)` at
boot. That means all of those devices initially belong to the host IPC
namespace. However, containers usually run in their own IPC namespace separate
from the host's. But when binder devices located in `/dev` are handed to
containers (e.g. with a bind-mount) the kernel driver will not know that these
devices are now used in a different IPC namespace since the driver is not IPC
namespace aware. This is not a serious technical issue but a serious conceptual
one. There should be a way to have per-IPC namespace binder devices.

#### Enter binderfs

To solve both problems we came up with a solution that I presented at the Linux
Plumbers Conference in Vancouver this year. There's a video of that
presentation available on Youtube:

<iframe width="560" height="315" src="https://www.youtube.com/embed/xMtDDEj-02c?start=1980" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Android binderfs is a tiny filesystem that allows users to dynamically allocate
binder devices, i.e. it allows to add and remove binder devices at runtime.
Which means it solves problem 1. Additionally, binder devices located in a new
binderfs instance are independent of binder devices located in another binderfs
instance. All binder devices in binderfs instances are also independent of the
binder devices allocated during boot specified in
`CONFIG_ANDROID_BINDER_DEVICES`. This means, binderfs solves problem 2.

Android binderfs can be mounted via:
```shell
mount -t binder binder /dev/binderfs
```
at which point a new instance of binderfs will show up at `/dev/binderfs`. In
a fresh instance of binderfs no binder devices will be present. There will only
be a `binder-control` device which serves as the request handler for binderfs:
```shell
root@edfu:~# ls -al /dev/binderfs/
total 0
drwxr-xr-x  2 root root      0 Jan 10 15:07 .
drwxr-xr-x 20 root root   4260 Jan 10 15:07 ..
crw-------  1 root root 242, 6 Jan 10 15:07 binder-control
```

#### binderfs: Dynamically Allocating a New binder Device

To allocate a new binder device in a binderfs instance a request needs to be
sent through the `binder-control` device node. A request is sent in the form of
an `ioctl(2)`. Here's an example program:
```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <linux/android/binder.h>
#include <linux/android/binderfs.h>

int main(int argc, char *argv[])
{
        int fd, ret, saved_errno;
        size_t len;
        struct binderfs_device device = { 0 };

        if (argc != 3)
                exit(EXIT_FAILURE);

        len = strlen(argv[2]);
        if (len > BINDERFS_MAX_NAME)
                exit(EXIT_FAILURE);

        memcpy(device.name, argv[2], len);

        fd = open(argv[1], O_RDONLY | O_CLOEXEC);
        if (fd < 0) {
                printf("%s - Failed to open binder-control device\n",
                       strerror(errno));
                exit(EXIT_FAILURE);
        }

        ret = ioctl(fd, BINDER_CTL_ADD, &device);
        saved_errno = errno;
        close(fd);
        errno = saved_errno;
        if (ret < 0) {
                printf("%s - Failed to allocate new binder device\n",
                       strerror(errno));
                exit(EXIT_FAILURE);
        }

        printf("Allocated new binder device with major %d, minor %d, and "
               "name %s\n", device.major, device.minor,
               device.name);

        exit(EXIT_SUCCESS);
}
```
What this program simply does is to open the `binder-control` device node and
sending a `BINDER_CTL_ADD` request to the kernel. Users of binderfs need to
tell the kernel which name the new binder device should get. By default a name
can only contain up to 256 chars including the terminating zero byte. The
struct which is used is:
```c
/**
 * struct binderfs_device - retrieve information about a new binder device
 * @name:   the name to use for the new binderfs binder device
 * @major:  major number allocated for binderfs binder devices
 * @minor:  minor number allocated for the new binderfs binder device
 *
 */
struct binderfs_device {
       char name[BINDERFS_MAX_NAME + 1];
       __u32 major;
       __u32 minor;
};
```
and is defined in `linux/android/binderfs.h`. Once the request is made via an
`ioctl(2)` passing a `struct binder_device` with the name to the kernel it will
allocate a new binder device and return the major and minor number of the new
device in the struct (This is necessary because binderfs allocated a major
device number dynamically at boot.). After the `ioctl(2)` returns there will be
a new binder device located under `/dev/binderfs` with the chosen name:
```shell
root@edfu:~# ls -al /dev/binderfs/
total 0
drwxr-xr-x  2 root root      0 Jan 10 15:19 .
drwxr-xr-x 20 root root   4260 Jan 10 15:07 ..
crw-------  1 root root 242, 0 Jan 10 15:19 binder-control
crw-------  1 root root 242, 1 Jan 10 15:19 my-binder
crw-------  1 root root 242, 2 Jan 10 15:19 my-binder1
```

#### binderfs: Deleting a binder Device

Deleting binder devices does not involve issuing another `ioctl(2)` request
through `binder-control`. They can be deleted via `unlink(2)`. This means that
the `rm(1)` tool can be used to delete them:
```shell
root@edfu:~# rm /dev/binderfs/my-binder1
root@edfu:~# ls -al /dev/binderfs/
total 0
drwxr-xr-x  2 root root      0 Jan 10 15:19 .
drwxr-xr-x 20 root root   4260 Jan 10 15:07 ..
crw-------  1 root root 242, 0 Jan 10 15:19 binder-control
crw-------  1 root root 242, 1 Jan 10 15:19 my-binder
```
Note that the `binder-control` device cannot be deleted since this would make
the binderfs instance unuseable. The `binder-control` device will be deleted
when the binderfs instance is unmounted and all references to it have been
dropped.

#### binderfs: Mounting Multiple Instances

Mounting another binderfs instance at a different location will create a new
and separate instance from all other binderfs mounts. This is identical to the
behavior of `devpts`, `tmpfs`, and also - even though never merged in the
kernel - `kdbusfs`:
```shell
root@edfu:~# mkdir binderfs1
root@edfu:~# mount -t binder binder binderfs1
root@edfu:~# ls -al binderfs1/
total 4
drwxr-xr-x  2 root   root        0 Jan 10 15:23 .
drwxr-xr-x 72 ubuntu ubuntu   4096 Jan 10 15:23 ..
crw-------  1 root   root   242, 2 Jan 10 15:23 binder-control
```
There is no `my-binder` device in this new binderfs instance since its devices
are not related to those in the binderfs instance at `/dev/binderfs`. This
means users can easily get their private set of binder devices.

#### binderfs: Mounting binderfs in User Namespaces

The Android binderfs filesystem can be mounted and used to allocate new binder
devices in user namespaces. This has the advantage that binderfs can be used in
unprivileged containers or any user-namespace-based sandboxing solution:
```shell
ubuntu@edfu:~$ unshare --user --map-root --mount
root@edfu:~# mkdir binderfs-userns
root@edfu:~# mount -t binder binder binderfs-userns/
root@edfu:~# The "bfs" binary used here is the compiled program from above
root@edfu:~# ./bfs binderfs-userns/binder-control my-user-binder
Allocated new binder device with major 242, minor 4, and name my-user-binder
root@edfu:~# ls -al binderfs-userns/
total 4
drwxr-xr-x  2 root root      0 Jan 10 15:34 .
drwxr-xr-x 73 root root   4096 Jan 10 15:32 ..
crw-------  1 root root 242, 3 Jan 10 15:34 binder-control
crw-------  1 root root 242, 4 Jan 10 15:36 my-user-binder
```

#### Kernel Patchsets

The [binderfs patchset][7] is merged upstream and will be available when Linux
5.0 gets released. There are a few outstanding patches that are currently
waiting in Greg's tree (cf. [binderfs: remove wrong kern_mount() call][8] and
[binderfs: make each binderfs mount a new instancechar-misc-linus][9]) and some
others are queued for the 5.1 merge window. But overall it seems to be in
decent shape.

[1]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/android/binder.c
[2]: https://source.android.com/devices/architecture/hidl/binder-ipc
[3]: https://elinux.org/Android_Binder
[4]: https://anbox.io/
[5]: https://github.com/lxc/lxc
[6]: https://lwn.net/Articles/618421/
[7]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/android/binderfs.c
[8]: https://git.kernel.org/pub/scm/linux/kernel/git/gregkh/char-misc.git/commit/?h=char-misc-linus&id=3fdd94acd50d607cf6a971455307e711fd8ee16e
[9]: https://git.kernel.org/pub/scm/linux/kernel/git/gregkh/char-misc.git/commit/?h=char-misc-linus&id=b6c770d7c9dc7185b17d53a9d5ca1278c182d6fa
