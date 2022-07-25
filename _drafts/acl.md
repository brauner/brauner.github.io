# POSIX ACLs

When creating or retrieving POSIX ACLs that store additional uids and gids via
`ACL_USER` or `ACL_GROUP` through the `setxattr()` and `getxattr()` system calls
various problems exist for stacked filesystems such as `overlayfs`.

But let's start with the seemingly simple status quo.

When running in a user namespace the uids and gids stored via `ACL_USER` and
`ACL_GROUP` attributes in POSIX ACLs need to be mapped into the caller's user
namespace. The caller's user namespace carries what we will refer to as the
"`caller_idmapping`" going forward.

This mapping or "translation" is performed by `posix_acl_fix_xattr_from_user()`
for `setxattr()` and by `posix_acl_fix_xattr_to_user()` for `getxattr()`.

The underlying helper called by `posix_acl_fix_xattr_{from,to}_user()` has a
pecular implemation that is important to understand in order to understand POSIX
ACLs with `ACL_USER` and `ACL_GROUP` attributes set:

(1) `posix_acl_fix_xattr_from_user()`

    caller_idmapping: u0:k10000000:r65536

    struct posix_acl_xattr_entry *entry;

    k10000004 = make_kuid(u0:k10000000:r65536 /* caller_idmapping */, u4 /* entry->e_id */);
    u10000004 = from_kuid(&init_user_ns, k10000004);

(2) `posix_acl_fix_xattr_to_user()`

    caller_idmapping: u0:k10000000:r65536

    struct posix_acl_xattr_entry *entry;

    k10000004 = make_kuid(&init_user_ns, u10000004 /* entry->e_id */);
    u4 = from_kuid(u0:k10000000:r65536 /* caller_idmapping */, k10000004);

It is important to note that in (1) and (2) the `init_user_ns` is used
independent of whether or not the underlying filesystem that the POSIX ACLs were
retrieved from has been mounted inside of a user namespace and thus have a
filesystem idmapping ("`fs_idmapping`").

Why is that relevant you may ask. Usually, when writing uids and gids passed in
from userspace to the filesystem involves translating the raw `uid_t` and
`gid_t` into a `kuid_t` and `kgid_t` right at the VFS boundary. Then they are
kept as `kuid_t` and `kgid_t` right until the moment where they need to be
written to disk.

For example, consider a regular `chown()` call:

    caller_idmapping: u0:k10000000:r65536
    fs_idmapping: u0:k10000000:r65536

    chown(/some/file, u4, u4);

Right when entering the VFS the kernel will map the `uid_t` and `gid_t` values
passed in from userspace into a `kuid_t` and `kgid_t`:


    k10000004 = make_kuid(u0:k10000000:r65536 /* fs_idmapping */, u4);
    k10000004 = make_kgid(u0:k10000000:r65536 /* fs_idmapping */, u4);

and store it in an appropriate kernel internal `struct iattr`. Once the inode is
changed and the change ultimately written back to disk the kernel will map the
`kuid_t` and `kgid_t` back into a `uid_t` and `gid_t` and write these values to
disk taking the `fs_idmapping` into account:

    u4 = from_kuid(u0:k10000000:r65536 /* fs_idmapping */, k10000004);
    u4 = from_kgid(u0:k10000000:r65536 /* fs_idmapping */, k10000004);

So right at the userspace to VFS boundary, the kernel will take care to map the
caller provided `uid_t` and `gid_t` into kernel internal `kuid_t` and `kgid_t`
representations. And the kernel will take care to never conflate `uid_t` and
`gid_t` values from there on. It will especially not translate them back into
`uid_t` and `gid_t` somewhere in the middle...

But for POSIX ACLs this isn't true. Going back to (1) we can see that the kernel
does indeed first translate the userspace `uid_t` and `gid_t` provided in
`ACL_USER` and `ACL_GROUP` attributes by mapping them via `make_kuid()` and
`make_kgid()` into `kuid_t` and `kgid_t` according to the `caller_idmapping``.

Suprisingly howerver, the `make_kuid()` and `make_kgid()` calls in (1) are
immediately followed by calls to `from_kuid()` and `from_kgid()` using the
`initial_idmapping` attached to `init_user_ns`.

And strangly all that this will achieve is an identity translation that turns
the `kuid_t` and `kgid_t` we just generated back into raw `uid_t` and `gid_t`
with the samve value as the `kuid_t` and `kgid_t` we just created. That seem
rather odd. Especially because we just noticed earlier that the kernel is
adamant about translating between userspace `uid_t` and `gid_t` into `kuid_t`
and `kgid_t` only at either the userspace to VFS or the VFS to on-disk boundary.

It contradicts the claim that we made earlier that the kernel will not translate
`kuid_t` and `kgid_t` into `uid_t` and `gid_t` somewhere in the middle of a
callchain. But we can see this weird behavior for POSIX ACLs all across the
callchains they are a part of.

The reason for this is simple and nasty. The VFS currently abuses the uapi
`struct posix_acl_xattr_entry` to transport `kuid_t` and `kgid_t` from the VFS
down to the filesystem and from the filesystem up to the VFS in the form of
userspace `uid_t` and `gid_t`. This has the consequence that the kernel
_intentionally_ breaks it's own type-safety requirements.

The only reason that the kernel uses the `initial_idmapping` in `from_kuid()`
and `from_kgid()` is to translate the just generated `kuid_t` and `kgid_t`s back
into `uid_t` and `gid_t` of the same value so that the target value can be
communicated to and from the filesystem without having to introduce an
additional kernel internal struct that would contain `kuid_t` and `kgid_t`
values and would perserve type safety. In other words, you're looking at a
classic hack instead of a proper redesign.

This has other consequences. For example, the callchains that POSIX ACLs are
involved in look extremely weird because the VFS _always_ uses the
`initial_idmapping` when translating between the uapi `struct
posix_acl_xattr_entry` and the kernel internal `struct posix_acl` __even if the
filesystem is mounted in a user namespace__ and thus would have an
`fs_idmapping` attached to it.

The cost of this is that it becomes very hard to understand what is going
especially when lacking proper documentation. It is extremely unclear why the
`fs_idmapping` is completely irrelevant even though the kernel translates between
`kuid_t` and `kgid_t` and `uid_t` and `gid_t` _multiple times_ in a standard
POSIX ACL callchain. This is bound to confuse developers and it is certain to
wreak havoc when bringing stacking filesystems like `overlayfs` into the mix.

## Before Linux v5.20

Let's look at a few POSIX ACL callchains before Linux v5.20 for various
filesystems. We will look at `xfs` and since the logic for `btrfs` and `ext4`
and various other filesystems is identical an `xfs` callchain can stand in for
all the other ones. None of the three filesystems can be mounted inside user
namespaces and so the only relevant `fs_idmapping` is the `initial_idmapping`.

But we will also look at `FUSE` which is the only filesystems that is mountable
inside of user namespaces while also being able to have a backing device.
Contrast this with `tmpfs` which is a filesystem that is mountable inside of
user namespace but has can never have a device backing it.

(3) `xfs`: `setxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k0:r4294967295 /* xfs can only be mounted in the initial userns */

     1 sys_setxattr()
     2 -> path_setxattr()
     3    -> setxattr()
     4       -> do_setxattr()
     5          -> setxattr_convert()
     6             -> posix_acl_fix_xattr_from_user()
     7                {
     8                         k10000004 = make_kuid(u0:k10000000:r65536 /* caller_idmapping */, u4);
    10                         u10000004 = from_kuid(&init_user_ns, k10000004);
    11               }
    12          -> vfs_setxattr() /* btrfs/ext4/xfs */
    13             -> __vfs_setxattr_locked()
    14                -> __vfs_setxattr_noperm()
    15                   -> __vfs_setxattr()
    16                      -> handler->set() == posix_acl_xattr_set()
    17                         -> posix_acl_from_xattr(&init_user_ns, ...)
    18                            {
    19                                     k10000004 = make_kuid(&init_user_ns, u10000004);
    20                            }
    21                            -> set_posix_acl()
    22                               -> posix_acl_valid(&init_user_ns /* fs_idmapping */, ...)
    23                                  {
    24                                           true = kuid_has_mapping(&init_user_ns /* fs_idmapping */, k10000004);
    25                                  }
    26                                  -> i_op->set_acl() == xfs_set_acl()  
    27                                     {
    28                                             u10000004 = from_kuid(&init_user_ns /* fs_idmapping */, k10000004);
    29                                     }
      
(4) `xfs`: `getxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k0:r4294967295 /* xfs can only be mounted in the initial userns */

     1 sys_getxattr()
     2 -> path_getxattr()
     3    -> getxattr()
     4       -> do_getxattr()
     5           -> vfs_getxattr() /* btrfs/ext4/xfs */
     6              -> __vfs_getxattr()
     7                 -> handler->get == posix_acl_xattr_get()
     8                    -> get_acl()
     9                       -> i_op->get_acl() == xfs_get_acl()
    10                          {
    11                                  k10000004 = make_kuid(&init_user_ns /* fs_idmapping */, u10000004);
    12                          }
    13                    -> posix_acl_to_xattr(&init_user_ns, ...)
    14                       {
    15                               u10000004 = from_kuid(&init_user_ns, k10000004);
    16                       }
    17           -> posix_acl_fix_xattr_to_user()
    18              {
    19                       k10000004 = make_kuid(&init_user_ns, u10000004);
    21                       u4 = from_kuid(u0:k10000000:r65536 /* caller_idmapping */, k10000004);
    22              }

(5) `FUSE` with `initial_idmapping`: `setxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k0:r4294967295  

     1 sys_setxattr()
     2 -> path_setxattr()
     3    -> setxattr()
     4       -> do_setxattr()
     5          -> setxattr_convert()
     6             -> posix_acl_fix_xattr_from_user()
     7                {
     8                         k10000004 = make_kuid(u0:k10000000:r65536 /* caller_idmapping */, u4);
    10                         u10000004 = from_kuid(&init_user_ns, k10000004);
    11                }
    12           -> vfs_setxattr() /* FUSE */
    13              -> __vfs_setxattr_locked()
    14                 -> __vfs_setxattr_noperm()
    15                    -> __vfs_setxattr()
    16                       -> handler->set() == posix_acl_xattr_set()
    17                          -> posix_acl_from_xattr(&init_user_ns, ...)
    18                             {
    19                                      k10000004 = make_kuid(&init_user_ns, u10000004);
    20                             }
    21                             -> set_posix_acl()
    22                                -> posix_acl_valid(&init_user_ns /* fs_idmapping */, ...)
    23                                   {
    24                                            true = kuid_has_mapping(&init_user_ns /* fs_idmapping */, k10000004);
    25                                   }
    26                                   -> i_op->set_acl() == fuse_set_acl()  
    27                                      -> posix_acl_to_xattr(&init_user_ns /* fs_idmapping */, ...)
    28                                         {
    29                                                  u10000004 = from_kuid(&init_user_ns /* fs_idmapping */, k10000004);
    30                                         }

(6) `FUSE` with `initial_idmapping`: `getxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k0:r4294967295  

     1 sys_getxattr()
     2 -> path_getxattr()
     3    -> getxattr()
     4       -> do_getxattr()
     5           -> vfs_getxattr() /* FUSE */
     6              -> __vfs_getxattr()
     7                 -> handler->get == posix_acl_xattr_get()
     8                    -> get_acl()
     9                       -> i_op->get_acl() == fuse_get_acl()
    10                          -> posix_acl_from_xattr(u0:k0:r4294967295 /* fs_idmapping */, ...) 
    11                             {
    12                                  k10000004 = make_kuid(u0:k0:r4294967295 /* fs_idmapping */, u10000004);
    13                             }
    14                    -> posix_acl_to_xattr(&init_user_ns, ...)
    15                       {
    16                               u10000004 = from_kuid(&init_user_ns /* fs_idmapping */, k10000004);
    17                       }
    18           -> posix_acl_fix_xattr_to_user()
    19              {
    20                       k10000004 = make_kuid(&init_user_ns, u10000004);
    22                       u4 = from_kuid(u0:k10000000:r65536 /* caller_idmapping */, k10000004);
    23              }

(7) `FUSE` with `fs_idmapping`: `setxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k10000000:r65536

     1 sys_setxattr()
     2 -> path_setxattr()
     3    -> setxattr()
     4       -> do_setxattr()
     5          -> setxattr_convert()
     6             -> posix_acl_fix_xattr_from_user()
     7                {
     8                         k10000004 = make_kuid(u0:k10000000:r65536 /* caller_idmapping */, u4);
    10                         u10000004 = from_kuid(&init_user_ns, k10000004);
    11                }
    12           -> vfs_setxattr() /* FUSE */
    13              -> __vfs_setxattr_locked()
    14                 -> __vfs_setxattr_noperm()
    15                    -> __vfs_setxattr()
    16                       -> handler->set() == posix_acl_xattr_set()
    17                          -> posix_acl_from_xattr(&init_user_ns, ...)
    18                             {
    19                                      k10000004 = make_kuid(&init_user_ns, u10000004);
    20                             }
    21                             -> set_posix_acl()
    22                                -> posix_acl_valid(u0:k10000000:r65536 /* fs_idmapping */, ...)
    23                                   {
    24                                            true = kuid_has_mapping(u0:k10000000:r65536 /* fs_idmapping */, k10000004);
    25                                   }
    26                                   -> i_op->set_acl() == fuse_set_acl()  
    27                                      -> posix_acl_to_xattr(u0:k10000000:r65536 /* fs_idmapping */, ...)
    28                                         {
    29                                                  u4 = from_kuid(u0:k10000000:r65536 /* fs_idmapping */, k10000004);
    30                                         }

(8) `FUSE` with `fs_idmapping`: `getxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k10000000:r65536

     1 sys_getxattr()
     2 -> path_getxattr()
     3    -> getxattr()
     4       -> do_getxattr()
     5           -> vfs_getxattr() /* FUSE */
     6              -> __vfs_getxattr()
     7                 -> handler->get == posix_acl_xattr_get()
     8                    -> get_acl()
     9                       -> i_op->get_acl() == fuse_get_acl()
    10                          -> posix_acl_from_xattr(u0:k10000000:r65536 /* fs_idmapping */, ...) 
    11                             {
    12                                  k10000004 = make_kuid(u0:k10000000:r65536 /* fs_idmapping */, u4);
    13                             }
    14                    -> posix_acl_to_xattr(&init_user_ns, ...)
    15                       {
    16                               u10000004 = from_kuid(&init_user_ns /* fs_idmapping */, k10000004);
    17                       }
    18           -> posix_acl_fix_xattr_to_user()
    19              {
    20                       k10000004 = make_kuid(&init_user_ns, u10000004);
    22                       u4 = from_kuid(u0:k10000000:r65536 /* caller_idmapping */, k10000004);
    23              }

(9) `overlayfs` with `initial_idmapping` on top of lower layer with `initial_idmapping`: `setxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k0:r4294967295 

     1 sys_setxattr()
     2 -> path_setxattr()
     3    -> setxattr()
     4       -> do_setxattr()
     5          -> setxattr_convert()
     6             -> posix_acl_fix_xattr_from_user()
     7                {
     8                         k10000004 = make_kuid(u0:k10000000:r65536 /* caller_idmapping */, u4);
    10                         u10000004 = from_kuid(&init_user_ns, k10000004);
    11                }
    12           -> vfs_setxattr() /* overlayfs */
    13              -> __vfs_setxattr_locked()
    14                 -> __vfs_setxattr_noperm()
    15                    -> __vfs_setxattr()
    16                       -> handler->set() == ovl_posix_acl_xattr_set()
    17                          -> posix_acl_from_xattr()
    18                             {
    19                                      k10000004 = make_kuid(&init_user_ns, u10000004)
    20                             }
    21                          -> ovl_xattr_set()
    22                             -> override_creds()
    23                             -> ovl_do_setxattr()
    24                                -> vfs_setxattr() /* btrfs/ext4/xfs */
    25                                   -> __vfs_setxattr_locked()
    26                                      -> __vfs_setxattr_noperm()
    27                                         -> __vfs_setxattr()
    28                                            -> handler->set() == posix_acl_xattr_set()
    29                                               -> posix_acl_from_xattr(&init_user_ns, ...)
    30                                                 {
    31                                                          k10000004 = make_kuid(&init_user_ns, u10000004);
    32                                                 }
    33                                                 -> set_posix_acl()
    34                                                    -> posix_acl_valid(&init_user_ns /* fs_idmapping */, ...)
    35                                                       {
    36                                                                true = kuid_has_mapping(&init_user_ns /* fs_idmapping */, k10000004);
    37                                                       }
    38                                                       -> i_op->set_acl() == xfs_set_acl()  
    39                                                          {
    40                                                                  u10000004 = from_kuid(&init_user_ns, k10000004);
    41                                                          }
    42                            -> revert_creds()

(10) `overlayfs` with `initial_idmapping` on top of lower layer with `initial_idmapping`: `getxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k0:r4294967295 

     1 sys_getxattr()
     2 -> path_getxattr()
     3    -> getxattr()
     4       -> do_getxattr()
     5           -> vfs_getxattr() /* overlayfs */
     6              -> __vfs_getxattr()
     7                 -> handler->get == ovl_posix_acl_xattr_get()
     8                    -> ovl_xattr_get()
     9                       -> vfs_getxattr() /* btrfs/ext4/xfs */
    10                          -> __vfs_getxattr()
    11                             -> handler->get == posix_acl_xattr_get()
    12                                -> get_acl()
    13                                   -> i_op->get_acl() == xfs_get_acl()
    14                                      {
    15                                              k4 = make_kuid(&init_user_ns /* fs_idmapping */, u4);
    16                                      }
    17                                -> posix_acl_to_xattr(&init_user_ns, ...)
    18                                   {
    19                                           u4 = from_kuid(&init_user_ns /* fs_idmapping */, k4);
    20                                   }
    21           -> posix_acl_fix_xattr_to_user()
    22              {
    23                       k4 = make_kuid(&init_user_ns, u4);
    25                       u-1 = from_kuid(u0:k10000000:r65536 /* caller_idmapping */, k4);
    26              }

(11) `overlayfs` with `fs_idmapping` on top of layer with `fs_idmapping`: `setxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k10000000:r65536 

     1 sys_setxattr()
     2 -> path_setxattr()
     3    -> setxattr()
     4       -> do_setxattr()
     5          -> setxattr_convert()
     6             -> posix_acl_fix_xattr_from_user()
     7                {
     8                         k10000004 = make_kuid(u0:k10000000:r65536 /* caller_idmapping */, u4);
    10                         u10000004 = from_kuid(&init_user_ns, k10000004);
    11                }
    12           -> vfs_setxattr() /* overlayfs */
    13              -> __vfs_setxattr_locked()
    14                 -> __vfs_setxattr_noperm()
    15                    -> __vfs_setxattr()
    16                       -> handler->set() == ovl_posix_acl_xattr_set()
    17                          -> posix_acl_from_xattr()
    18                             {
    19                                      k10000004 = make_kuid(&init_user_ns, u10000004)
    20                             }
    21                          -> ovl_xattr_set()
    22                             -> override_creds()
    23                             -> ovl_do_setxattr()
    24                                -> vfs_setxattr() /* FUSE */
    25                                   -> vfs_setxattr()
    26                                      -> __vfs_setxattr_locked()
    27                                         -> __vfs_setxattr_noperm()
    28                                            -> __vfs_setxattr()
    29                                               -> handler->set() == posix_acl_xattr_set()
    30                                                  -> posix_acl_from_xattr(&init_user_ns, ...)
    31                                                     {
    32                                                              k10000004 = make_kuid(&init_user_ns, u10000004);
    33                                                     }
    34                                                     -> set_posix_acl()
    35                                                        -> posix_acl_valid(u0:k10000000:r65536 /* fs_idmapping */, ...)
    36                                                           {
    37                                                                    true = kuid_has_mapping(u0:k10000000:r65536 /* fs_idmapping */, k10000004);
    38                                                           }
    39                                                           -> i_op->set_acl() == fuse_set_acl()  
    40                                                              -> posix_acl_to_xattr(u0:k10000000:r65536 /* fs_idmapping */, ...)
    41                                                                 {
    42                                                                          u4 = from_kuid(u0:k10000000:r65536 /* fs_idmapping */, k10000004);
    43                                                                 }
    44                             -> revert_creds()

(12) `overlayfs` with `fs_idmapping` on top of layer with `fs_idmapping`: `getxattr()`

    mnt_idmapping:    u0:k0:r4294967295
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k10000000:r65536 

     1 sys_getxattr()
     2 -> path_getxattr()
     3    -> getxattr()
     4       -> do_getxattr()
     5           -> vfs_getxattr() /* overlayfs */
     6              -> __vfs_getxattr()
     7                 -> handler->get == ovl_posix_acl_xattr_get()
     8                    -> ovl_xattr_get()
     9                       -> vfs_getxattr() /* FUSE */
    10                          -> __vfs_getxattr()
    11                             -> handler->get == posix_acl_xattr_get()
    12                                -> get_acl()
    13                                   -> i_op->get_acl() == fuse_get_acl()
    14                                      -> posix_acl_from_xattr(u0:k10000000:r65536 /* fs_idmapping */, ...) 
    15                                         {
    16                                              k10000004 = make_kuid(u0:k10000000:r65536 /* fs_idmapping */, u4);
    17                                         }
    18                                -> posix_acl_to_xattr(&init_user_ns, ...)
    19                                   {
    20                                           u10000004 = from_kuid(&init_user_ns /* fs_idmapping */, k10000004);
    21                                   }
    22           -> posix_acl_fix_xattr_to_user()
    23              {
    24                       k10000004 = make_kuid(&init_user_ns, u10000004);
    26                       u4 = from_kuid(u0:k10000000:r65536 /* caller_idmapping */, k10000004);
    27              }


As we've said `xfs` and the others cannot be mounted with an idmapping and thus
the relevant `fs_idmapping` is the `initial_idmapping`. This has the
consequence that some implementation details are more difficult to spot. 

For example, in the `setxattr()` callchain (3) line 28 and in the `getxattr()`
callchain (4) line 11 the `initial_idmapping` is used because `xfs` can only be
mounted in `init_user_ns`. But for other filesystems the `fs_idmapping` can be
different from the `initial_idmapping`.

If `FUSE` is mounted without an `fs_idmapping` the logic is identical to `xfs`.
If it is mounted with an `fs_idmapping` the logic is different on the filesystem
level where the `fs_idmapping` is taken into account in the `setxattr()`
callchain (7) line 29 and in the `getxattr()` callchain (8) line 12.

Note however, how even in the case where an `fs_idmapping` is used as in (5) and
(6) VFS callchains such as

    posix_acl_xattr_set()
    -> posix_acl_from_xattr(&init_user_ns, ...)

or

    posix_acl_xattr_get()
    -> posix_acl_to_xattr(&init_user_ns, ...)

__always__ use the `initial_idmapping` as can be seen in (7) line 19 and (8)
line 16.

For `setxattr()` we pointed out earlier that in
`posix_acl_fix_xattr_from_user()` the type system is subverted by turning a
`kuid_t` and `kgid_t` into a `uid_t` and `gid_t` sending what should be a
`kuid_t` and `kgid_t` down as `uid_t` and `gid_t`.

In the `setxattr()` callchain (7) line 19 we can see that in the call to
`posix_acl_from_xattr()` the subversion is reverted by abusing the
`initial_idmapping` again and turning the `uid_t` and `gid_t` into a `kuid_t`
and `kgid_t` again.

For `getxattr()` the type safety violation happens not at the userspace to VFS
boundary but at the device to VFS boundary. The `getxattr()` callchain (8) line
16 abuses the `initial_idmapping` to turn a `kuid_t` and `kgid_t` into a `uid_t`
and `gid_t` so it can be stored in the uapit `struct posix_acl_xattr_entry` and
sent back up to the VFS to userspace boundary. This happens __in the middle of a
VFS callchain__.

The subversion is reverted again in `posix_acl_fix_xattr_to_user()` by abusing
the `initial_idmapping` again to turn the `uid_t` and `gid_t` back into a
`kuid_t` and `kgid_t`. The `kuid_t` and `kgid_t` just generated then immediately
gets mapped into `uid_t` and `gid_t` in the `caller_idmapping` in order to
report it to userspace.

This is a whole lot of subtlety and ripe for confusion and bugs.

On `overlayfs` the callchains become more complicated because `overlayfs` is a
stacking filesystems. What this means that certain callchains will be hit twice.
For example, the VFS encapsulates some core filesystem functionality in helpers
prefixed with `vfs_` such as `vfs_setxattr()` or `vfs_getxattr(). You can spot
those helpers in the callchains above.

A stacking filesystem such as `overlayfs` will need to call these helpers in
order to perform operations on the filesystem it is stacked upon. For example,
if you mount an `overlayfs` filesystem:

```sh
mount -t overlay overlay -o lowerdir=/lower_layer_1:lower_layer_2,    \
                            upperdir=/writable_layer/upper,         \
                            workdir=/writable_layer/work            \
                            /merge
```

and have created a file `/merge/file` in there setting a POSIX ACL on this file
via `setxattr()` will generate a nested callchain. So blanking out all
distracting details of the `setxattr()` callchain from a callchain like (9) or
(11) above we get:

     1 sys_setxattr()
     2
     3
     4
     5
     6
     7
     8
    10
    11
    12           -> vfs_setxattr() /* overlayfs */
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24                                -> vfs_setxattr() /* xfs or FUSE */

meaning that the `vfs_setxattr()` helper is called once for the `overlayfs`
layer and a second time for the underlying filesystem such as `xfs` or `FUSE`.

You can see that the subtle type confusions are very problematic because now
they have to be reasoned about in scenarios where we are dealing with a stacking
filesystem. The type confusion is kept around for an even longer time and passed
into ever more complicated and deep callchains.

This hack didn't matter for a long time until someone came a long and
implemented a new feature.

## Idmapped Mounts

Idmapped mounts work by allowing to attach idmappings to mounts. You might have
noticed these idmappings above as `mnt_idmapping` before the callchains we lined
out.

In essence, idmapped mounts function like a temporary and localized `chown()`
operation where the ownership changes are tied to the lifetime of a mount.

During permission checking and when creating new filesystem objects or reporting
ownership information the `mnt_idmapping` needs to be taken into account.

Say you have a file that is stored on disk as being owned by `uid_t` and `gid_t`
`u0`. Now you mount that filesystem in a user namespace with the `fs_idmapping`
`u0:k10000000:r65536`. When the filesystem initializes ` struct inode` for the
file and fills in ownership information via:

```c
static inline void i_uid_write(struct inode *inode, uid_t uid)
{
	inode->i_uid = make_kuid(i_user_ns(inode), uid);
}
```

this comes down to:

	k10000000 = make_kuid(u0:k10000000:r65536 /* fs_idmapping */, u0);

and means the `struct inode` for the file will contain `k10000000` in
`inode->i_uid` and `inode->i_gid`.

Now when an idmapped mount is the `mnt_idmapping` needs to be taken into account
when ownership information is needed. This comes down to undoing the
`fs_idmapping` and applying the `mnt_idmapping`. The details around this are
documented in _Documentation/filesystems/idmappings.rst` in the Linux kernel
repository so I won't repeat it all here.

Currently no filesystems mountable inside of user namespaces support idmapped
mounts so our explanation becomes a little simpler since the `fs_idmapping` is
always the `initial_idmapping`.

Let's look at an idmapped mount created for an `xfs` filesystem. Since `xfs` can
only be mounted with the `initial_idmapping` the value stored as the ownership
of a file on disk is identical to the value stored in the `inode->i_uid` and
`inode->i_gid` in `struct inode`.

They are separate types of course, as the `inode->i_uid` and `inode->i_gid`
members have type `kuid_t` and `kgid_t` while the device ownership is expressed
in terms of `uid_t` and `gid_t`. In any case, for this example `inode->i_uid`
and `inode->i_gid` for the file will contain `k0`:

	k0 = make_kuid(&init_user_ns /* fs_idmapping */, u0);

Now assume we have created an idmapped mount for some directory or the whole
filesystem with the idmapping `u0:k10000000:r65536`. This means that the kernel
will remap `inode->i_uid` and `inode->i_gid` on the fly during permission
checking. This logic is epressed in `make_vfsuid()` and `make_vfsgid()`:

    v10000000 = make_vfsuid(u0:k10000000:r65536 /* mnt_idmapping */, &init_user_ns /* fs_idmapping */, k0)

and to reverse:

    k0 = from_vfsuid(u0:k10000000:r65536 /* mnt_idmapping */, &init_user_ns /* fs_idmapping */, v10000000)

The difference to `make_kuid()` and `make_kgid()` is that the input isn't
`uid_t` or `gid_t` but a `kuid_t` and `kgid_t` and the output is a `vfsuid_t`
and `vfsgid_t`. This is a dedicated type introduced to ensure that `kuid_t` and
`kgid_t` can never be conflated with a `vfsuid_t` and `vfsgid_t`. In other
words, we ensure that a `mnt_idmapping` generates a new ownership type that only
ever appears in the VFS and can't be accidently passed to `from_kuid()` and
`from_kgid()` or `make_kuid()` and `make_kgid()`.

## Idmapped Mounts and POSIX ACLs

In order to support idmapped mounts all relevant callchains had to be updated to
take the `mnt_idmapping` into account. This includes all relevant POSIX ACL
codepaths.

So the translation helpers in (1) and (2) were adapated to:

(12) `posix_acl_fix_xattr_from_user()`

    caller_idmapping: u0:k10000000:r65536
    mnt_idmapping:    k0:v10000000:r65536

    struct posix_acl_xattr_entry *entry;

    k10000004 = make_kuid(u0:k10000000:r65536 /* caller_idmapping */, u4 /* entry->e_id */);

    v10000005 = make_vfsuid(&init_user_ns, &init_user_ns, k10000004);
           k4 = from_vfsuid(k0:v10000000:r65536 /* mnt_idmapping */, &init_user_ns, v10000004);

           u4 = from_kuid(&init_user_ns, k4);

(13) `posix_acl_fix_xattr_to_user()`

    caller_idmapping: u0:k10000000:r65536
    mnt_idmapping:    k0:v10000000:r65536

    struct posix_acl_xattr_entry *entry;

           k4 = make_kuid(&init_user_ns, u4 /* entry->e_id */);

    v10000005 = make_vfsuid(k0:v10000000:r65536 /*mnt_idmapping, &init_user_ns, k4);
    k10000004 = from_vfsuid(&init_user_ns, /* mnt_idmapping */, &init_user_ns, v10000004);

    u4 = from_kuid(u0:k10000000:r65536 /* caller_idmapping */, k10000004);

The consequence of the earlier type confusion is that we need to play the same
game on idmapped mounts that we played earlier. We need to abuse the
`initial_idmapping` again to subvert the type system.

The first problem is that `make_vfsuid()` and `make_vfsgid()` and
`from_vfsuid()` and `from_vfsgid()` are conceptually tied to the `fs_idmapping`.

For example, `make_vfsuid()` and `make_vfsgid()` need to undo the `fs_idmapping`
for VFS objects such as `struct inode` `inode->i_uid` and `inode->i_gid` and the
apply the `mnt_idmapping`.

The `from_vfsuid()` and `from_vfsgid()` helpers are the inverse of
`make_vfsuid()` and `make_vfsgid()` and thus need to undo the `mnt_idmapping`
before applying the `fs_idmapping`.

The helpers elegantly express the dependency between the `fs_idmapping` and the
`mnt_idmapping` neatly and thus suggest that the `fs_idmapping` be used whenever
they are called.

The problem now is that the type safety subversion in `ACL_USER` and `ACL_GROUP`
POSIX ACLs now becomes even more problematic. First, because the developer needs
to be aware that the `fs_idmapping` is completely irrelevant in
`posix_acl_fix_xattr_from_user()` and `posix_acl_fix_xattr_to_user()` and thus
is also irrelevant for `make_vfsuid()`, `make_vfsgid()`, `from_vfsuid()`, and
`from_vfsgid()`. That itself is already a feat because the developer needs to be
aware of the callchain pecularities of POSIX ACLs. That problem would not exist
if the type passed down into the filesystem would be a `kuid_t` and `kgid_t`
right from the start.

Second, the `mnt_idmapping` is a property of the VFS that is tied to a mount
created for a specific filesystem. So the `mnt_idmapping` cannot be taken into
account for `ACL_USER` and `ACL_GROUP` POSIX ACLs in some early place of the
callchains. Since they are a property of each layer in a filesystem stack
generated by `overlayfs` they need to be taken into account at specific places
in the callchain.

So consider the callchain we abbreviated earlier to make the stacking of
`vfs_setxattr()` calls for `overlayfs` more obvious:

     1 sys_setxattr()
     2
     3
     4
     5
     6
     7
     8
    10
    11
    12           -> vfs_setxattr() /* overlayfs */
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24                                -> vfs_setxattr() /* xfs or FUSE */

The interesting question is where the `mnt_idmapping` is currently applied:

     1 sys_setxattr()
     2
     3
     4       -> do_setxattr()
     5          -> setxattr_convert()
     6             -> posix_acl_fix_xattr_from_user() /* translation step */
     7                  
     8                                                                                                
    10                                                                          
    11                  
    12           -> vfs_setxattr() /* overlayfs */
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24                                -> vfs_setxattr() /* xfs or FUSE */

We can see that the `mnt_idmapping` is applied outside of `vfs_setxattr()` in
`do_setxattr()`.

Let's assume we have a filesystem that has a POSIX ACL for `ACL_USER` set with
`u4` stored on the backing device and we create an idmapped mount with
`k0:v10000000:r65536` so that a caller with the `caller_idmapping`
`u0:k10000000:r65536` will see the POSIX ACL as being owned by `u4` when
retrieving it. Currently this does not work:

(14)

    mnt_idmapping:    k0:v10000000:r65536
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k0:r4294967295 

     1 sys_getxattr()
     2 -> path_getxattr()
     3    -> getxattr()
     4       -> do_getxattr()
     5           -> vfs_getxattr() /* overlayfs */
     6              -> __vfs_getxattr()
     7                 -> handler->get == ovl_posix_acl_xattr_get()
     8                    -> ovl_xattr_get()
     9                       -> vfs_getxattr() /* btrfs/ext4/xfs */
    10                          -> __vfs_getxattr()
    11                             -> handler->get == posix_acl_xattr_get()
    12                                -> get_acl()
    13                                   -> i_op->get_acl() == xfs_get_acl()
    14                                      {
    15                                              k4 = make_kuid(&init_user_ns /* fs_idmapping */, u4);
    16                                      }
    17                                -> posix_acl_to_xattr(&init_user_ns, ...)
    18                                   {
    19                                           u4 = from_kuid(&init_user_ns /* fs_idmapping */, k4);
    20                                   }
    21           -> posix_acl_fix_xattr_to_user()
    22              {
    23                       k4 = make_kuid(&init_user_ns, u4);
    24                       v4 = make_vfsuid(&init_user_ns /* mnt_idmapping */, &init_user_ns, k4);
    25                       k4 = from_vfsuid(&init_user_ns, /* mnt_idmapping */, &init_user_ns, v4);
    26                       u-1 = from_kuid(u0:k10000000:r65536 /* caller_idmapping */, k4);
    27              }

We can see in (14) line 27 that the translation yield `u-1` which just means
that the `caller_idmapping` doesn't contain a mapping for `k4`. And the reason
is that the `mnt_idmapping` never got taken into account which would have ensure
that `k10000004` would've been returned which does have a mapping in the
`caller_idmapping`.

The reason why the `mnt_idmapping` wasn't taken into account is that the the
translation happens only in the top layer of the filesystem stack. In other
words, the translation happens for the `overlayfs` filesystem and the
`overlayfs` filesystem has not been on an idmapped mount (Side-note: In fact,
`overlayfs` can only be mounted on top of idmapped mounts but it does not
support the creation of idmapped mount and probably never will.).

The translation step involving the `mnt_idmapping` needs to happen in
`vfs_getxattr()` and `vfs_setxattr()` so that the VFS can take the
`mnt_idmapping` into account for each filesystem it is called upon.

Starting with Linux v5.20 this is exactly what happens. Instead of performing
the `mnt_idmapping` translations in `posix_acl_fix_xattr_from_user()` and
`posix_acl_fix_xattr_to_user()` the `mnt_idmapping` is taken into account in
`vfs_setxattr()` and `vfs_getxattr()`:

(15)

    mnt_idmapping:    k0:v10000000:r65536
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k0:r4294967295 

     1 sys_setxattr()
     2 -> path_setxattr()
     3    -> setxattr()
     4       -> do_setxattr()
     5          -> setxattr_convert()
     6             -> posix_acl_fix_xattr_from_user()
     7                {
     8                         k10000004 = make_kuid(u0:k10000000:r65536 /* caller_idmapping */, u4);
     9                         u10000004 = from_kuid(&init_user_ns, k10000004);
    10                }
    11           -> vfs_setxattr() /* overlayfs */
    12              -> posix_acl_setxattr_idmapped_mnt()
    13                 {
    14                         k10000004 = make_kuid(&init_user_ns, u10000004);
    15                         v10000004 = make_vfsuid(&init_user_ns, &init_user_ns, k10000004);
    16                         k10000004 = from_vfsuid(&init_user_ns, &init_user_ns, v10000004);
    17                         u10000004 = from_kuid(&init_user_ns, k10000004);
    18                 }
    19              -> __vfs_setxattr_locked()
    20                 -> __vfs_setxattr_noperm()
    21                    -> __vfs_setxattr()
    22                       -> handler->set() == ovl_posix_acl_xattr_set()
    23                          -> posix_acl_from_xattr()
    24                             {
    25                                      k10000004 = make_kuid(&init_user_ns, u10000004)
    26                             }
    27                          -> ovl_xattr_set()
    28                             -> override_creds()
    29                             -> ovl_do_setxattr()
    30                                -> vfs_setxattr() /* btrfs/ext4/xfs */
    31                                   -> posix_acl_setxattr_idmapped_mnt()
    32                                      {
    33                                              k10000004 = make_kuid(&init_user_ns, k10000004);
    34                                              v10000004 = make_vfsuid(&init_user_ns, &init_user_ns, k10000004);
    35                                                     k4 = from_vfsuid(k0:v10000000:r65536 /* mnt_idmapping */, &init_user_ns, v10000004);
    36                                                     u4 = from_kuid(&init_user_ns, k4);
    37                                      }
    38                                   -> __vfs_setxattr_locked()
    39                                      -> __vfs_setxattr_noperm()
    40                                         -> __vfs_setxattr()
    41                                            -> handler->set() == posix_acl_xattr_set()
    42                                               -> posix_acl_from_xattr(&init_user_ns, ...)
    43                                                 {
    44                                                          k4 = make_kuid(&init_user_ns, u4);
    45                                                 }
    46                                                 -> set_posix_acl()
    47                                                    -> posix_acl_valid(&init_user_ns /* fs_idmapping */, ...)
    48                                                       {
    49                                                                true = kuid_has_mapping(&init_user_ns /* fs_idmapping */, k4);
    50                                                       }
    51                                                       -> i_op->set_acl() == xfs_set_acl()  
    52                                                          {
    53                                                                  u4 = from_kuid(&init_user_ns, k4);
    54                                                          }
    55                            -> revert_creds()

(16)

    mnt_idmapping:    k0:v10000000:r65536
    caller_idmapping: u0:k10000000:r65536
    fs_idmapping:     u0:k0:r4294967295 

     1 sys_getxattr()
     2 -> path_getxattr()
     3    -> getxattr()
     4       -> do_getxattr()
     5           -> vfs_getxattr() /* overlayfs */
     6              -> posix_acl_getxattr_idmapped_mnt()
     7                 {
     8                         k10000004 = make_kuid(&init_user_ns, u10000004);
     9                         v10000004 = make_vfsuid(&init_user_ns, &init_user_ns, k10000004);
    10                         k10000004 = from_vfsuid(&init_user_ns, &init_user_ns, v10000004);
    11                         u10000004 = from_kuid(&init_user_ns, k10000004);
    12                 }
    13              -> __vfs_getxattr()
    14                 -> handler->get == ovl_posix_acl_xattr_get()
    15                    -> ovl_xattr_get()
    16                       -> vfs_getxattr() /* btrfs/ext4/xfs */
    17                          -> __vfs_getxattr()
    18                             -> handler->get == posix_acl_xattr_get()
    19                                -> get_acl()
    20                                   -> i_op->get_acl() == xfs_get_acl()
    21                                      {
    22                                              k4 = make_kuid(&init_user_ns /* fs_idmapping */, u4);
    23                                      }
    24                                -> posix_acl_to_xattr(&init_user_ns, ...)
    25                                   {
    26                                           u4 = from_kuid(&init_user_ns /* fs_idmapping */, k4);
    27                                   }
    28                          -> posix_acl_getxattr_idmapped_mnt()
    29                             {
    30                                     k4 = make_kuid(&init_user_ns, u4);
    31                                     v10000004 = make_vfsuid(k0:v10000000:r65536 /* mnt_idmapping */, &init_user_ns, k4);
    32                                     k10000004 = from_vfsuid(&init_user_ns, &init_user_ns, v10000004);
    33                                     u10000004 = from_kuid(&init_user_ns, k10000004);
    34                             }
    35           -> posix_acl_fix_xattr_to_user()
    36              {
    37                       k10000004 = make_kuid(&init_user_ns, u10000004);
    38                       u10000004 = from_kuid(u0:k10000000:r65536 /* caller_idmapping */, k10000004);
    39              }

A bug like this seems pretty obvious in hindsight but in fact it really isn't
when the type system is flouted as it did. And looking closely you will see that
this type safety issue is haunting is even now as we constantly need to abuse
the `initial_idmapping` just for the sake of convertion between the different
types and to be able to continue abusing the uapi `struct posix_acl_xattr_entry`
to pass `kuid_t` and `kgid_t` as `uid_t` and `gid_t` values.

## Confusing `fs_idmapping` relevance

A while ago we received a few questions about the role of the `fs_idmapping` for
POSIX ACLs as we were fixing the `overlayfs` POSIX ACL handling on top of
idmapped layers where I explained that the `mnt_idmapping` needs to be handled
in `vfs_getxattr()`/`vfs_setxattr()`.

The question was why `posix_acl_fix_xattr_from_user()` and
`posix_acl_fix_xattr_to_user()` couldn't just be moved into `vfs_setxattr()` and
`vfs_getxattr()` and simply taking the `fs_idmapping` into account in these
helpers.

This questions shows how dangerous the type subversion is. The fact is that the
`fs_idmapping` is __entirely irrelevant__ for `posix_acl_fix_xattr_from_user()`
and `posix_acl_fix_xattr_to_user()` as we've seen. The reason why it __seems__
relevant is that we convert between a `kuid_t`/`kgid_t` and `uid_t`/`gid_t` but
that conversion is entirely based on a hack to pass `kuid_t`/`kgid_t` in a uapi
`struct posix_acl_xattr_entry` as `uid_t`/`gid_t`.

For a stacking filesystem the `fs_idmapping` of the top layer needs to be
transparent with respect to the ownership of things such as `inode->i_uid`,
`inode->i_gid`, `ACL_USER`/`ACL_GROUP` and POSIX ACLs and filesystem
capabilities.

The general idea is that right at the userspace to VFS boundary any
`uid_t`/`gid_t` passed in from userspace is immediately turned into a
`kuid_t`/`kgid_t`. The `kuid_t`/`kgid_t` is kept and only translated into a
`uid_t`/`gid_t` at the VFS to backing device boundary, i.e., when a write to a
backing device occurs.

A great example is a simple `chown()` operation. In `chown_common()` the passed
in `uid_t`/`gid_t` is turned into `kuid_t`/`kgid_t` in the `caller_idmapping`.
The `notify_change()` helper is called to determine whether the caller is
allowed to perform the ownership change. Part of that is validating via
`kuid_has_mapping()`/`kgid_has_mapping()` that the requested `kuid_t`/`kgid_t`
has a mapping in the `fs_idmapping`. If it does we know we can translate the
`kuid_t`/`kgid_t` back into `uid_t`/`gid_t` when writing to the backing device
later on. But nowhere do we subvert the type system.

For `overlayfs` the `chown()` operation is exactly similar just that permission
checking is performed twice as `notify_change()` is called once on the
`overlayfs` layer and once on the lower layer. But note that the requested
`kuid_t`/`kgid_t` never takes the `fs_idmapping` or `caller_idmapping` of the
mounter of the `overlayfs` filesystem into account.

Similarly, when we report ownership information of a file the lower layer will
fill in `struct kstat` with the ownership information based on the
`fs_idmapping` and `mnt_idmapping` of the lower layer. The `fs_idmapping` and
`caller_idmapping` of the `overlayfs` mounter are entirely irrelevant for this.

This is very crucial to notice as taking the `fs_idmapping` or
`caller_idmapping` for `overlayfs` into account would mean that `overlayfs`
changes the ownership of the underlying filesystem objects when mounted inside
of a user namespace. That would very likely be a security issue.

But this is exactly what moving `posix_acl_fix_xattr_from_user()` and
`posix_acl_fix_fix_xattr_to_user()` into `vfs_getxattr()` and `vfs_setxattr()`
would do. If these helpers were relocated and also would allow to take the
`fs_idmapping` into account then POSIX ACL ownership would be changed by the
`overlayfs` layer for the underlying filesystem.
