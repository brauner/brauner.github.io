---
layout: post
title: 'On The Way To LXC 3.0: Splitting Out Templates And Language Bindings'
---

![alt text](https://linuxcontainers.org/static/img/containers.png)

Hey everyone,

This is another update about the development of `LXC 3.0`.

We are currently in the process of moving various parts of `LXC` out of the
main [LXC](https://github.com/lxc/lxc) repository and into separate
repositories.

#### Splitting Out The Language Bindings For `Lua` And `Python 3`

The lua language bindings will be moved into the new
[lua-lxc](https://github.com/lxc/lua-lxc) repository and the Python 3 bindings
to the new [python3-lxc](https://github.com/lxc/python3-lxc) repository.
This is in line with other language bindings like Python 2 (see
[python2-lxc](https://github.com/lxc/python2-lxc)) that were always kept out of
tree.

#### Splitting Out The Legacy Template Build System

A big portion of the `LXC` templates will be moved to the new
[lxc-templates](https://github.com/lxc/lxc-templates) repository.
`LXC` used to maintain simple shell scripts to build container images from for
a lot of distributions including `CentOS`, `Fedora`, `ArchLinux`, `Ubuntu`,
`Debian` and a lot of others. While the shell scripts worked well for a long
time they suffered from the problem that they were often different in terms of
coding style, the arguments that they expected to be passed, and the features
they supported. A lot of things these shells scripts did when creating an image
is not needed any more. For example, most distros nowadays provide a custom
cloud image suitable for containers and virtual machines or at least provide
their own tooling to build clean new images from scratch. Another problem we
saw was that security and maintenance for the scripts was not sufficient. This
is why we decided to come up with a simple yet elegant replacement for the
template system that would still allow users to build custom `LXC` and `LXD`
container images for the distro of their choice. So the templates will be
replaced by [distrobuilder](https://github.com/lxc/distrobuilder) as the
preferred way to build `LXC` and `LXD` images locally.
[distrobuilder](https://github.com/lxc/distrobuilder) is a project my colleague
[Thomas](https://github.com/monstermunchkin) is currently working on. It aims
to be a very simple Go project focussed on letting you easily build full system
container images by either **using the official cloud image** if one is
provided by the distro or by **using the respective distro's recommended
tooling** (e.g. `debootstrap` for `Debian` or `pacman` for `ArchLinux`). It
aims to be declarative, using the same set of options for all distributions
while having extensive validation code to ensure everything that's downloaded
is properly validated.

After this cleanup only four `POSIX` shell compliant templates will remain in
the main `LXC` repository:

- **`busybox`**

This is a very minimal template which can be used to setup a `busybox`
container. As long as the `busybox` binary is found you can always built
yourself a very minimal privileged or unprivileged system or application
container image; no networking or any other dependencies required. All you need
to do is:
```
lxc-create c3 -t busybox
```

[![asciicast](https://asciinema.org/a/165788.png)](https://asciinema.org/a/165788)

- **`download`**

This template lets you download pre-built images from our image servers. This
is likely what currently most users are using to create unprivileged
containers.

- **`local`**

This is a new template which consumes standard `LXC` and `LXD` system
container images. A container can be created with:
```
lxc-create c1 -t local -- --metadata /path/to/meta.tar.xz --fstree /path/to/rootfs.tar.xz
```

where the `--metadata` flag needs to point to a file containing the metadata
for the container. This is simply the standard `meta.tar.xz` file that comes
with any pre-built `LXC` container image. The `--fstree` flag needs to point to
a filesystem tree. Creating a container is then just:

[![asciicast](https://asciinema.org/a/165783.png)](https://asciinema.org/a/165783)

- **`oci`**

This is the template which can be used to download and run OCI containers.
Using it is as simple as:
```
lxc-create c2 -t oci -- --url docker://alpine
```
Here's another asciicast:

[![asciicast](https://asciinema.org/a/165784.png)](https://asciinema.org/a/165784)


