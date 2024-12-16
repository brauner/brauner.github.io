---
layout: post
title: Listing all mounts in all mount namespaces
---

# Introduction

A little while we added a new api for retrieving information about mounts.

# `listmount(2)` and `statmount(2)`

To make it easier to interact with mounts the `listmount(2)` and `statmount(2)` system calls were introduced in Linux `v6.9`.
They both allow interacting with mounts through the new `64 bit` mount id (unsigned) that is assigned to each mount on the system.
The new mount id isn't recycled and is unique for the lifetime of the system whereas the old mount id was recycled frequently and maxed out at `INT_MAX`.
To differentiate the new and old mount id the new mount id starts at `INT_MAX + 1`.

Both `statmount(2)` and `listmount(2)` take a `struct mnt_id_req` as their first argument:

```c
/*
 * Structure for passing mount ID and miscellaneous parameters to statmount(2)
 * and listmount(2).
 *
 * For statmount(2) @param represents the request mask.
 * For listmount(2) @param represents the last listed mount id (or zero).
 */
struct mnt_id_req {
	__u32 size;
	__u32 spare;
	__u64 mnt_id;
	__u64 param;
	__u64 mnt_ns_id;
};
```

The struct is versioned by size and thus extensible.

## `statmount(2)`

`statmount()` allows to retrieved detailed information about a mount.
The mount to retrieve information about can be specified in `mnt_id_req->mnt_id`.
The information is supposed to be retrieved must be specified in `mnt_id_req->param`.

```c
struct statmount {
	__u32 size;              /* Total size, including strings */
	__u32 mnt_opts;          /* [str] Options (comma separated, escaped) */
	__u64 mask;              /* What results were written */
	__u32 sb_dev_major;      /* Device ID */
	__u32 sb_dev_minor;
	__u64 sb_magic;          /* ..._SUPER_MAGIC */
	__u32 sb_flags;          /* SB_{RDONLY,SYNCHRONOUS,DIRSYNC,LAZYTIME} */
	__u32 fs_type;           /* [str] Filesystem type */
	__u64 mnt_id;            /* Unique ID of mount */
	__u64 mnt_parent_id;     /* Unique ID of parent (for root == mnt_id) */
	__u32 mnt_id_old;        /* Reused IDs used in proc/.../mountinfo */
	__u32 mnt_parent_id_old;
	__u64 mnt_attr;          /* MOUNT_ATTR_... */
	__u64 mnt_propagation;   /* MS_{SHARED,SLAVE,PRIVATE,UNBINDABLE} */
	__u64 mnt_peer_group;    /* ID of shared peer group */
	__u64 mnt_master;        /* Mount receives propagation from this ID */
	__u64 propagate_from;    /* Propagation from in current namespace */
	__u32 mnt_root;          /* [str] Root of mount relative to root of fs */
	__u32 mnt_point;         /* [str] Mountpoint relative to current root */
	__u64 mnt_ns_id;         /* ID of the mount namespace */
	__u32 fs_subtype;        /* [str] Subtype of fs_type (if any) */
	__u32 sb_source;         /* [str] Source string of the mount */
	__u32 opt_num;           /* Number of fs options */
	__u32 opt_array;         /* [str] Array of nul terminated fs options */
	__u32 opt_sec_num;       /* Number of security options */
	__u32 opt_sec_array;     /* [str] Array of nul terminated security options */
	__u64 __spare2[46];
	char str[];              /* Variable size part containing strings */
};

/*
 * @mask bits for statmount(2)
 */
#define STATMOUNT_SB_BASIC       0x00000001U /* Want/got sb_... */
#define STATMOUNT_MNT_BASIC      0x00000002U /* Want/got mnt_... */
#define STATMOUNT_PROPAGATE_FROM 0x00000004U /* Want/got propagate_from */
#define STATMOUNT_MNT_ROOT       0x00000008U /* Want/got mnt_root  */
#define STATMOUNT_MNT_POINT      0x00000010U /* Want/got mnt_point */
#define STATMOUNT_FS_TYPE        0x00000020U /* Want/got fs_type */
#define STATMOUNT_MNT_NS_ID      0x00000040U /* Want/got mnt_ns_id */
#define STATMOUNT_MNT_OPTS       0x00000080U /* Want/got mnt_opts */
#define STATMOUNT_FS_SUBTYPE     0x00000100U /* Want/got fs_subtype */
#define STATMOUNT_SB_SOURCE      0x00000200U /* Want/got sb_source */
#define STATMOUNT_OPT_ARRAY      0x00000400U /* Want/got opt_... */
#define STATMOUNT_OPT_SEC_ARRAY  0x00000800U /* Want/got opt_sec... */
```

## `listmount(2)`
`listmount(2)` allows to (recursively) retrieve the list of child mounts of the provided mount.
The mount whose children to list is specified in `mnt_id_req->mnt_id`.
For convenience it can be set to `LSTMT_ROOT` to start listing mounts from the rootfs mount.

A nice feature of `listmount(2)` is its ability to iterate through all mounts in a mount namespace.
For example, a buffer for `100` mounts ids is passed to `listmount(2)` but mount namespace contains more than `100` mounts.
`listmount(2)` will retrieve `100` mounts.
Afterwards the `mnt_id_req->param` can be set to the last mount id returned in the previous request.
`listmount(2)` will return the next mount after the last mount.

`listmount(2)` obviously also allows iterating through subtrees.
This is as simple as setting `mnt_id_req->mnt_id` to the mount whose children to retrieve.

By default `listmount(2)` return will return earlier mounts before later mounts.
This can be changed by passing `LISTMOUNT_REVERSE` to `listmount(2)`.
`LISTMOUNT_REVERSE` will cause it to list later mounts before earlier mounts.

# Listing mounts in other mount namespaces

Both `listmount(2)` and `stamount(2)` by default operate on mounts in the caller's mount namespace.
But both support operating on another mount namespace.
Either the unique `64 bit` mount namespace id can be specified in `mnt_id_req->mnt_ns_id` or a mount namespace file descriptor can be set in `mnt_id_req->spare`.

In order to list mounts in another mount namespace the caller must have `CAP_SYS_ADMIN` in the owning user namespace of the mount namespace.

## Listing mount namespaces

The mount namespace id can be retrieved via the new `NS_MNT_GET_INFO` nsfs `ioctl(2)`.
It takes a `struct mnt_ns_info` and fills it in:

```c
struct mnt_ns_info {
        __u32 size;
        __u32 nr_mounts;
        __u64 mnt_ns_id;
};
```

The mount namespace id will be returned in `mnt_ns_info->mnt_ns_id`.
In addition to this it will also return the number of mounts in the mount namespace in `mnt_ns_info->nr_mounts`.
This can be used to size the buffer for `listmount(2)`.

This is accompanied by two other `nsfs` ioctls.
`ioctl(fd_mntns, NS_MNT_GET_NEXT)` returns the mount namespace after `@fd_mntns` and `ioctl(fd_mntns, NS_MNT_GET_PREV)` returns the mount namespace before `@fd_mntns`.
These two ioctls allow to iterate through all mount namespaces in a backwards or forwards manner.
Both also optionally take a `struct mnt_ns_info` argument to retrieve information about the mount namespace.

All three ioctls are available in Linux `v6.12`.

# Conclusion

Taken all these pieces together a suitably privileged process is able to iterate through all mounts in all mount namespaces.
Here is a (dirty) sample program to illustrate how this can be done.
Note that the program below assumes that the caller is in the initial mount and user namespace.
When listing mount namespaces a mount namespace will only be listed if the caller has `CAP_SYS_ADMIN` in the owning user namespace otherwise it will be skipped.

```c
// SPDX-License-Identifier: GPL-2.0-or-later
// Copyright (c) 2024 Christian Brauner <brauner@kernel.org>

#define _GNU_SOURCE
#include <errno.h>
#include <limits.h>
#include <linux/types.h>
#include <stdio.h>
#include <sys/ioctl.h>
#include <sys/syscall.h>

#define die_errno(format, ...)                                             \
	do {                                                               \
		fprintf(stderr, "%m | %s: %d: %s: " format "\n", __FILE__, \
			__LINE__, __func__, ##__VA_ARGS__);                \
		exit(EXIT_FAILURE);                                        \
	} while (0)

/* Get the id for a mount namespace */
#define NS_GET_MNTNS_ID _IO(0xb7, 0x5)
/* Get next mount namespace. */

struct mnt_ns_info {
	__u32 size;
	__u32 nr_mounts;
	__u64 mnt_ns_id;
};

#define MNT_NS_INFO_SIZE_VER0 16 /* size of first published struct */

/* Get information about namespace. */
#define NS_MNT_GET_INFO _IOR(0xb7, 10, struct mnt_ns_info)
/* Get next namespace. */
#define NS_MNT_GET_NEXT _IOR(0xb7, 11, struct mnt_ns_info)
/* Get previous namespace. */
#define NS_MNT_GET_PREV _IOR(0xb7, 12, struct mnt_ns_info)

#define PIDFD_GET_MNT_NAMESPACE _IO(0xFF, 3)

#ifndef __NR_listmount
#define __NR_listmount 458
#endif

#ifndef __NR_statmount
#define __NR_statmount 457
#endif

/* @mask bits for statmount(2) */
#define STATMOUNT_SB_BASIC		0x00000001U /* Want/got sb_... */
#define STATMOUNT_MNT_BASIC		0x00000002U /* Want/got mnt_... */
#define STATMOUNT_PROPAGATE_FROM	0x00000004U /* Want/got propagate_from */
#define STATMOUNT_MNT_ROOT		0x00000008U /* Want/got mnt_root  */
#define STATMOUNT_MNT_POINT		0x00000010U /* Want/got mnt_point */
#define STATMOUNT_FS_TYPE		0x00000020U /* Want/got fs_type */
#define STATMOUNT_MNT_NS_ID		0x00000040U /* Want/got mnt_ns_id */
#define STATMOUNT_MNT_OPTS		0x00000080U /* Want/got mnt_opts */

#define STATX_MNT_ID_UNIQUE 0x00004000U /* Want/got extended stx_mount_id */

struct statmount {
	__u32 size;
	__u32 mnt_opts;
	__u64 mask;
	__u32 sb_dev_major;
	__u32 sb_dev_minor;
	__u64 sb_magic;
	__u32 sb_flags;
	__u32 fs_type;
	__u64 mnt_id;
	__u64 mnt_parent_id;
	__u32 mnt_id_old;
	__u32 mnt_parent_id_old;
	__u64 mnt_attr;
	__u64 mnt_propagation;
	__u64 mnt_peer_group;
	__u64 mnt_master;
	__u64 propagate_from;
	__u32 mnt_root;
	__u32 mnt_point;
	__u64 mnt_ns_id;
	__u64 __spare2[49];
	char str[];
};

struct mnt_id_req {
	__u32 size;
	__u32 spare;
	__u64 mnt_id;
	__u64 param;
	__u64 mnt_ns_id;
};

#define MNT_ID_REQ_SIZE_VER1 32 /* sizeof second published struct */

#define LSMT_ROOT 0xffffffffffffffff /* root mount */

static int __statmount(__u64 mnt_id, __u64 mnt_ns_id, __u64 mask,
		       struct statmount *stmnt, size_t bufsize,
		       unsigned int flags)
{
	struct mnt_id_req req = {
		.size		= MNT_ID_REQ_SIZE_VER1,
		.mnt_id		= mnt_id,
		.param		= mask,
		.mnt_ns_id	= mnt_ns_id,
	};

	return syscall(__NR_statmount, &req, stmnt, bufsize, flags);
}

static struct statmount *sys_statmount(__u64 mnt_id, __u64 mnt_ns_id,
				       __u64 mask, unsigned int flags)
{
	size_t bufsize = 1 << 15;
	struct statmount *stmnt = NULL, *tmp = NULL;
	int ret;

	for (;;) {
		tmp = realloc(stmnt, bufsize);
		if (!tmp)
			goto out;

		stmnt = tmp;
		ret = __statmount(mnt_id, mnt_ns_id, mask, stmnt, bufsize, flags);
		if (!ret)
			return stmnt;

		if (errno != EOVERFLOW)
			goto out;

		bufsize <<= 1;
		if (bufsize >= UINT_MAX / 2)
			goto out;
	}

out:
	free(stmnt);
	return NULL;
}

static ssize_t sys_listmount(__u64 mnt_id, __u64 last_mnt_id, __u64 mnt_ns_id,
			     __u64 list[], size_t num, unsigned int flags)
{
	struct mnt_id_req req = {
		.size		= MNT_ID_REQ_SIZE_VER1,
		.mnt_id		= mnt_id,
		.param		= last_mnt_id,
		.mnt_ns_id	= mnt_ns_id,
	};

	return syscall(__NR_listmount, &req, list, num, flags);
}

int main(int argc, char *argv[])
{
#define LISTMNT_BUFFER 10
	__u64 list[LISTMNT_BUFFER], last_mnt_id = 0;
	int ret, pidfd, fd_mntns;
	struct mnt_ns_info info = {};

	pidfd = sys_pidfd_open(getpid(), 0);
	if (pidfd < 0)
		die_errno("pidfd_open failed");

	fd_mntns = ioctl(pidfd, PIDFD_GET_MNT_NAMESPACE, 0);
	if (fd_mntns < 0)
		die_errno("ioctl(PIDFD_GET_MNT_NAMESPACE) failed");

	ret = ioctl(fd_mntns, NS_MNT_GET_INFO, &info);
	if (ret < 0)
		die_errno("ioctl(NS_GET_MNTNS_ID) failed");

	printf("Listing %u mounts for mount namespace %llu\n",
	       info.nr_mounts, info.mnt_ns_id);
	for (;;) {
		ssize_t nr_mounts;
next:
		nr_mounts = sys_listmount(LSMT_ROOT, last_mnt_id,
					  info.mnt_ns_id, list, LISTMNT_BUFFER,
					  0);
		if (nr_mounts <= 0) {
			int fd_mntns_next;

			printf("Finished listing %u mounts for mount namespace %llu\n\n",
			       info.nr_mounts, info.mnt_ns_id);
			fd_mntns_next = ioctl(fd_mntns, NS_MNT_GET_NEXT, &info);
			if (fd_mntns_next < 0) {
				if (errno == ENOENT) {
					printf("Finished listing all mount namespaces\n");
					exit(0);
				}
				die_errno("ioctl(NS_MNT_GET_NEXT) failed");
			}
			close(fd_mntns);
			fd_mntns = fd_mntns_next;
			last_mnt_id = 0;
			printf("Listing %u mounts for mount namespace %llu\n",
			       info.nr_mounts, info.mnt_ns_id);
			goto next;
		}

		for (size_t cur = 0; cur < nr_mounts; cur++) {
			struct statmount *stmnt;

			last_mnt_id = list[cur];

			stmnt = sys_statmount(last_mnt_id, info.mnt_ns_id,
					      STATMOUNT_SB_BASIC |
					      STATMOUNT_MNT_BASIC |
					      STATMOUNT_MNT_ROOT |
					      STATMOUNT_MNT_POINT |
					      STATMOUNT_MNT_NS_ID |
					      STATMOUNT_MNT_OPTS |
					      STATMOUNT_FS_TYPE, 0);
			if (!stmnt) {
				printf("Failed to statmount(%llu) in mount namespace(%llu)\n",
				       last_mnt_id, info.mnt_ns_id);
				continue;
			}

			printf("mnt_id:\t\t%llu\nmnt_parent_id:\t%llu\nfs_type:\t%s\nmnt_root:\t%s\nmnt_point:\t%s\nmnt_opts:\t%s\n\n",
			       stmnt->mnt_id,
			       stmnt->mnt_parent_id,
			       stmnt->str + stmnt->fs_type,
			       stmnt->str + stmnt->mnt_root,
			       stmnt->str + stmnt->mnt_point,
			       stmnt->str + stmnt->mnt_opts);
			free(stmnt);
		}
	}

	exit(0);
}
```

# Contributing fixes, improvements, or corrections to this post

If you have fixes, improvements, or corrections feel free to email them to me or simply open a pull request against [the repository for this blog](https://github.com/brauner/brauner.github.io).
