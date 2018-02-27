---
layout: post
title: 'On The Way To LXC 3.0: Splitting Out Templates And Language Bindings'
---

![alt text](https://linuxcontainers.org/static/img/containers.png)

Hey everyone,

This is another update about the development of `LXC 3.0`.

We are currently in the process of moving various parts of `LXC` out of the
main [LXC](https://github.com/lxc/lxc) repository and into separate
repositories. The lua language bindings will be moved into the new
[lua-lxc](https://github.com/lxc/lua-lxc) repository and the Python 3 bindings
to the new [python3-lxc](https://github.com/lxc/python3-lxc) repository.

Furthermore, various `LXC` templates will be moved to the new
[lxc-templates](https://github.com/lxc/lxc-templates) repository as they will
soon be replaced by [distrobuilder](https://github.com/lxc/distrobuilder) as
the preferred way to build `LXC` and `LXD` images locally.
This is a project my colleague [Thomas Hipp](https://github.com/monstermunchkin)
is currently working on. It aims to be a very simple GO project focussed on
letting you easily build full system container images by either **using the
official cloud image** if one is provided by the distro or by **using the
respective distro's recommended tooling** (e.g. `debootstrap` for `Debian` or
`pacman` for `ArchLinux`).

After this cleanup only four `POSIX` shell compliant templates will remain in
the main `LXC` repository:

- **`lxc-busybox`**

This is a very minimal template which can be used to setup a `busybox`
container. As long as the `busybox` binary is found you can always built
yourself a very minimal privileged or unprivileged system or application
container image; no networking or any other dependencies required. All you need
to do is:
```
lxc-create c3 -t busybox
```

[![asciicast](https://asciinema.org/a/165788.png)](https://asciinema.org/a/165788)

- **`lxc-download`**

This template lets you download pre-built images from our image servers. This
is likely what currently most users are using to create unprivileged
containers.

- **`lxc-local`**

This is a new template which consumes standard `LXC` and `LXD` system
container images. A container can be created with:
```
lxc-create c1 -t local -- --metadata </path/to/meta.tar.xz> --fstree </path/to/rootfs.tar.xz>
```

where the `--metadata` flag needs to point to a file containing the metadata
for the container. This is simply the standard `meta.tar.xz` file that comes
with any pre-built `LXC` container image. The `--fstree` flag needs to point to
a filesystem tree. Creating a container is then just:

[![asciicast](https://asciinema.org/a/165783.png)](https://asciinema.org/a/165783)

- **`lxc-oci`**

This is the template which can be used to download and run OCI containers.
Using it is as simple as:
```
lxc-create c2 -t oci -- --url docker://alpine
```
Here's another asciicast:

[![asciicast](https://asciinema.org/a/165784.png)](https://asciinema.org/a/165784)


