---
layout: post
title: The Seccomp Notifier - Cranking up the crazy with bpf()
---

In my last article I looked at the [seccomp notifier](https://people.kernel.org/brauner/the-seccomp-notifier-new-frontiers-in-unprivileged-container-development) in detail and how it allows us to make unprivileged containers way more capable (Sorry, kernel joke.). This is the (very) crazy (but very short) sequel. (Sorry Jon, no novella this time. :))

Last time I mentioned two new features that we had landed:

1. [Retrieving file descriptors from another task via `pidfd_getfd()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=83fa805bcbfc53ae82eedd65132794ae324798e5)
2. [Injection file descriptors via the new `SECCOMP_IOCTL_NOTIF_ADDFD` ioctl on the seccomp notifier](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9ecc6ea491f0c0531ad81ef9466284df260b2227)

The 2. feature just landed in the merge window for `v5.9`. So what better time than now to boot a `v5.9` pre-rc1 kernel and play with the new features.

I said that these features make it possible to intercept syscalls that return file descriptors or that pass file descriptors to the kernel. Syscalls that come to mind are `open()`, `connect()`, `dup2()`, but also `bpf()`.
People that read the first blogpost might not have realized how crazy^serious one can get with these two new features so I thought it be a good exercise to illustrate it. And what better victim than `bpf()`.

As we know, `bpf()` and unprivileged containers don't get along too well. But that doesn't need to be the case. For the demo you're about to see I enabled LXD to supervise the `bpf()` syscalls for tasks running in unprivileged containers. We will intercept the `bpf()` syscalls for the `BPF_PROG_LOAD` command for `BPF_PROG_TYPE_CGROUP_DEVICE` program types and the `BPF_PROG_ATTACH`, and `BPF_PROG_DETACH` commands for the `BPF_CGROUP_DEVICE` attach type. This allows a nested unprivileged container to load its own device profile in the cgroup2 hierarchy.

This is just a tiny glimpse into how this can be used and extended. ;) The pull request for LXD is already up [here](https://github.com/lxc/lxd/pull/7743). Let's see if the rest of the team thinks I'm going crazy. :)

[![asciicast](https://asciinema.org/a/352181.svg)](https://asciinema.org/a/352191)
