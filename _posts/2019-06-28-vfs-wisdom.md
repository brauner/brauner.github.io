---
layout: post
title: 'Linux Kernel VFSisms'
---

#### Introduction

This is intended as a collection of helpful knowledge bits around Linus Kernel
VFS internals. It mostly contains (hopefully) useful bits and pieces I picked
up while working on the Linux kernel and talking to VFS maintainers or
high-profile contributors.

#### `ksys_close()`

Should never be used. One of the major reasons being that it is too easy to get
wrong.

#### On creating and installing new file descriptors

A file descriptor should only be installed past every possible point of
failure. Specifically for a syscall the file descriptor should be installed
right before returning to userspace.
Consider the function `anon_inode_getfd()`. This functions creates and installs
a new file descriptor for a task. Hence, by the rule given above it should only
ever be called when the syscall cannot fail anymore in any other way then by
failing `anon_inode_getfd()`.

For all other cases the rule is to **reserve** a file descriptor but defer the
**installation** of the file descriptor past the last point of failure. Note,
that installing an file descriptor itself is not an operation that can fail.

Back to the anonymous inode example: Instead of calling `anon_inode_getfd()`
callers who need a file descriptor before the last point of failure should
reserve a file descriptor, call `anon_inode_getfile()` and then defer the
`fd_install()` until after the last point of failure. Here is a concrete
example blessed by Al Viro:
```c
	if (clone_flags & CLONE_PIDFD) {
		/* reserve a new file descriptor */
		retval = get_unused_fd_flags(O_RDWR | O_CLOEXEC);
		if (retval < 0)
			goto bad_fork_free_pid;

		pidfd = retval;

		/* get file to associate with file descriptor */
		pidfile = anon_inode_getfile("[pidfd]", &pidfd_fops, pid,
					      O_RDWR | O_CLOEXEC);
		if (IS_ERR(pidfile)) {
			put_unused_fd(pidfd);
			retval = ERR_PTR(pidfile);
			goto bad_fork_free_pid;
		}
		get_pid(pid);	/* held by pidfile now */

		/* place file descriptor in buffer accessible for userspace */
		retval = put_user(pidfd, parent_tidptr);
		if (retval)
			goto bad_fork_put_pidfd;
	}

	/* a lot more code that can fail somehow */

	/* Let kill terminate clone/fork in the middle */
	if (fatal_signal_pending(current)) {
		retval = -EINTR;
		goto bad_fork_cancel_cgroup;
	}

	/* past the last point of failure */
	if (pidfile)
		fd_install(pidfd, pidfile);
```
