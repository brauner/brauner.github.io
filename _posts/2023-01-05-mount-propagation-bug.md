---
layout: post
title: An excursion into a mount propagation bug
---

# Introduction

At the end of 2022 we received a bug report about the following splat:

```
[  115.848393] BUG: kernel NULL pointer dereference, address: 0000000000000010
[  115.848967] #PF: supervisor read access in kernel mode
[  115.849386] #PF: error_code(0x0000) - not-present page
[  115.849803] PGD 0 P4D 0
[  115.850012] Oops: 0000 [#1] PREEMPT SMP PTI
[  115.850354] CPU: 0 PID: 15591 Comm: mount Not tainted 6.1.0-rc7 #3
[  115.850851] Hardware name: innotek GmbH VirtualBox/VirtualBox, BIOS
VirtualBox 12/01/2006
[  115.851510] RIP: 0010:propagate_one.part.0+0x7f/0x1a0
[  115.851924] Code: 75 eb 4c 8b 05 c2 25 37 02 4c 89 ca 48 8b 4a 10
49 39 d0 74 1e 48 3b 81 e0 00 00 00 74 26 48 8b 92 e0 00 00 00 be 01
00 00 00 <48> 8b 4a 10 49 39 d0 75 e2 40 84 f6 74 38 4c 89 05 84 25 37
02 4d
[  115.853441] RSP: 0018:ffffb8d5443d7d50 EFLAGS: 00010282
[  115.853865] RAX: ffff8e4d87c41c80 RBX: ffff8e4d88ded780 RCX: ffff8e4da4333a00
[  115.854458] RDX: 0000000000000000 RSI: 0000000000000001 RDI: ffff8e4d88ded780
[  115.855044] RBP: ffff8e4d88ded780 R08: ffff8e4da4338000 R09: ffff8e4da43388c0
[  115.855693] R10: 0000000000000002 R11: ffffb8d540158000 R12: ffffb8d5443d7da8
[  115.856304] R13: ffff8e4d88ded780 R14: 0000000000000000 R15: 0000000000000000
[  115.856859] FS:  00007f92c90c9800(0000) GS:ffff8e4dfdc00000(0000)
knlGS:0000000000000000
[  115.857531] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  115.858006] CR2: 0000000000000010 CR3: 0000000022f4c002 CR4: 00000000000706f0
[  115.858598] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[  115.859393] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[  115.860099] Call Trace:
[  115.860358]  <TASK>
[  115.860535]  propagate_mnt+0x14d/0x190
[  115.860848]  attach_recursive_mnt+0x274/0x3e0
[  115.861212]  path_mount+0x8c8/0xa60
[  115.861503]  __x64_sys_mount+0xf6/0x140
[  115.861819]  do_syscall_64+0x5b/0x80
[  115.862117]  ? do_faccessat+0x123/0x250
[  115.862435]  ? syscall_exit_to_user_mode+0x17/0x40
[  115.862826]  ? do_syscall_64+0x67/0x80
[  115.863133]  ? syscall_exit_to_user_mode+0x17/0x40
[  115.863527]  ? do_syscall_64+0x67/0x80
[  115.863835]  ? do_syscall_64+0x67/0x80
[  115.864144]  ? do_syscall_64+0x67/0x80
[  115.864452]  ? exc_page_fault+0x70/0x170
[  115.864775]  entry_SYSCALL_64_after_hwframe+0x63/0xcd
[  115.865187] RIP: 0033:0x7f92c92b0ebe
[  115.865480] Code: 48 8b 0d 75 4f 0c 00 f7 d8 64 89 01 48 83 c8 ff
c3 66 2e 0f 1f 84 00 00 00 00 00 90 f3 0f 1e fa 49 89 ca b8 a5 00 00
00 0f 05 <48> 3d 01 f0 ff ff 73 01 c3 48 8b 0d 42 4f 0c 00 f7 d8 64 89
01 48
[  115.866984] RSP: 002b:00007fff000aa728 EFLAGS: 00000246 ORIG_RAX:
00000000000000a5
[  115.867607] RAX: ffffffffffffffda RBX: 000055a77888d6b0 RCX: 00007f92c92b0ebe
[  115.868240] RDX: 000055a77888d8e0 RSI: 000055a77888e6e0 RDI: 000055a77888e620
[  115.868823] RBP: 0000000000000000 R08: 0000000000000000 R09: 0000000000000001
[  115.869403] R10: 0000000000001000 R11: 0000000000000246 R12: 000055a77888e620
[  115.869994] R13: 000055a77888d8e0 R14: 00000000ffffffff R15: 00007f92c93e4076
[  115.870581]  </TASK>
[  115.870763] Modules linked in: nft_fib_inet nft_fib_ipv4
nft_fib_ipv6 nft_fib nft_reject_inet nf_reject_ipv4 nf_reject_ipv6
nft_reject nft_ct nft_chain_nat nf_nat nf_conntrack nf_defrag_ipv6
nf_defrag_ipv4 ip_set rfkill nf_tables nfnetlink qrtr snd_intel8x0
sunrpc snd_ac97_codec ac97_bus snd_pcm snd_timer intel_rapl_msr
intel_rapl_common snd vboxguest intel_powerclamp video rapl joydev
soundcore i2c_piix4 wmi fuse zram xfs vmwgfx crct10dif_pclmul
crc32_pclmul crc32c_intel polyval_clmulni polyval_generic
drm_ttm_helper ttm e1000 ghash_clmulni_intel serio_raw ata_generic
pata_acpi scsi_dh_rdac scsi_dh_emc scsi_dh_alua dm_multipath
[  115.875288] CR2: 0000000000000010
[  115.875641] ---[ end trace 0000000000000000 ]---
[  115.876135] RIP: 0010:propagate_one.part.0+0x7f/0x1a0
[  115.876551] Code: 75 eb 4c 8b 05 c2 25 37 02 4c 89 ca 48 8b 4a 10
49 39 d0 74 1e 48 3b 81 e0 00 00 00 74 26 48 8b 92 e0 00 00 00 be 01
00 00 00 <48> 8b 4a 10 49 39 d0 75 e2 40 84 f6 74 38 4c 89 05 84 25 37
02 4d
[  115.878086] RSP: 0018:ffffb8d5443d7d50 EFLAGS: 00010282
[  115.878511] RAX: ffff8e4d87c41c80 RBX: ffff8e4d88ded780 RCX: ffff8e4da4333a00
[  115.879128] RDX: 0000000000000000 RSI: 0000000000000001 RDI: ffff8e4d88ded780
[  115.879715] RBP: ffff8e4d88ded780 R08: ffff8e4da4338000 R09: ffff8e4da43388c0
[  115.880359] R10: 0000000000000002 R11: ffffb8d540158000 R12: ffffb8d5443d7da8
[  115.880962] R13: ffff8e4d88ded780 R14: 0000000000000000 R15: 0000000000000000
[  115.881548] FS:  00007f92c90c9800(0000) GS:ffff8e4dfdc00000(0000)
knlGS:0000000000000000
[  115.882234] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  115.882713] CR2: 0000000000000010 CR3: 0000000022f4c002 CR4: 00000000000706f0
[  115.883314] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[  115.883966] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
```

The bug could be reproduced, albeit unreliably, by running the mount propagation test of the [LTP](https://github.com/linux-test-project/ltp) testsuite in a loop while simultaneously creating various network namespaces with `ip netns` command.
When we started debugging this it turned out that the interesting aspect of `ip netns` for this bug was that it persisted a network namespace by bind-mounting it in a separate mount namespace.

It turned out that the reliability of the reproducer could be significantly increased by using

```
unshare --mount --propagation=unchanged -- mount --make-rslave /
```

while using the [a specific test](https://github.com/linux-test-project/ltp/blob/af98698067f706feeb1729e038eef9aefc12760c/testcases/kernel/fs/fs_bind/bind/fs_bind24.sh#L4) of the LTP mount propagation testsuite, modifying it slightly so that we loop around `mount` and `umount` in the script.

In previous years Seth Forshee and I had reported and fixed issues in the mount propagation code.
Each of these bugs had been hard to understand but only required trivial patches in order to be fixed.
My expectation was no different for this bug.

The mount propagation code uses the now obsolete 'slave' and 'master' concepts to express dependency relationships.
As the data structures themselves use these terms they are used here as well.

## Basic mount propagation concepts

The `propagate_mnt()` function handles mount propagation when creating mounts.
It propagates a source mount tree `@source_mnt` to all applicable nodes of a destination propagation tree headed by the destination mount `@dest_mnt`.

While fixing this bug we've gotten confused multiple times due to unclear terminology or missing concepts.
So we'll start this with some clarifications:

* The terms 'master' or 'peer' denote a shared mount.
  A shared mount belongs to a peer group.

* A peer group is a set of shared mounts that propagate to each other.
  They are identified by a peer group id. The peer group id is available in `@shared_mnt->mnt_group_id`.
  Shared mounts within the same peer group have the same peer group id.
  The peers in a peer group can be reached via `@shared_mnt->mnt_share`.

* The terms 'slave mount' or 'dependent mount' denote a mount that receives propagation from a peer in a peer group.
  Thus, shared mounts may have slave mounts and slave mounts have shared mounts as their master.
  Slave mounts of a given peer in a peer group are listed on that peers slave list available at `@shared_mnt->mnt_slave_list`.

* The term 'master mount' denotes a mount in a peer group.
  In other words, it denotes a shared mount or a peer mount in a peer group.
  The term 'master mount' - or 'master' for short - is mostly used when talking in the context of slave mounts that receive propagation from a master mount.
  A master mount of a slave identifies the closest peer group a slave mount receives propagation from.
  The master mount of a slave can be identified via `@slave_mount->mnt_master`.
  Different slaves may point to different masters in the same peer group.

* Multiple peers in a peer group can have non-empty `->mnt_slave_list`s.
  Non-empty `->mnt_slave_lists` of peers don't intersect.
  Consequently, to ensure all slave mounts of a peer group are visited the `->mnt_slave_list`s of all peers in a peer group have to be walked.

* Slave mounts point to a peer in the closest peer group they receive propagation from via `@slave_mnt->mnt_master` (see above).
  Together with these peers they form a propagation group (see below).
  The closest peer group can thus be identified through the peer group id `@slave_mnt->mnt_master->mnt_group_id` of the peer/master that a slave mount receives propagation from.

* A shared-slave mount is a slave mount to a peer group `pg1` while also a peer in another peer group `pg2`.
  This simply means that a peer group may receive propagation from another peer group.

  If a peer group `pg2` is a slave to another peer group `pg1` then all peers in peer group `pg2` point to the same peer in peer group `pg1` via `->mnt_master`.
  So all peers in peer group `pg2` appear on the same `->mnt_slave_list` of a peer in `pg1`.
  So they cannot be slaves to different peer groups or even different peers in the same peer group.

* A pure slave mount is a slave to a peer group but is not a peer in another peer group.

* A propagation group denotes the set of mounts consisting of a single peer group `pg1` and all slave mounts and shared-slave mounts that point to a peer in that peer group via `->mnt_master`.
  This means all slave mounts such that `@slave_mnt->mnt_master->mnt_group_id` is equal to `@shared_mnt->mnt_group_id`.

  The concept of a propagation group makes it easier to talk about a single propagation level in a propagation tree.

  For example, in `propagate_mnt()` the immediate peers of `@dest_mnt` and all slaves of `@dest_mnt`'s peer group form a propagation group `propg1`.
  So a shared-slave mount that is a slave in `propg1` and that is a peer in another peer group `pg2` forms another propagation group `propg2` together with all slaves that point to that shared-slave mount in their `->mnt_master`.

* A propagation tree refers to all mounts that receive propagation starting from a specific shared mount.

  For example, for `propagate_mnt()` the destination mount `@dest_mnt` is the start of a propagation tree.
  The propagation tree encompasses all mounts that receive propagation from `@dest_mnt`'s peer group down to the leafs.

## The bug in one sentence

The `propagate_mnt()` function contains a bug where it fails to terminate at peers of `@source_mnt` when searching for a copy of `@source_mnt` which is a suitable master for a new copy mounted on top of a slave in the destination propagation tree, causing a NULL dereference.

## The impact of the bug

Once the mechanics of the bug are understood it's easy to trigger.
Because of unprivileged user namespaces it is available to unprivileged users.
When the bug triggers `namespace_lock()` is held which is a read-write semaphore that needs to be held in a host of scenarios.
The gist is that once the bug has been triggered most interactions with the filesystem become impossible.
The kernel is effectively deadlocked.

Since we're not attackers we're not sure whether this bug can be exploited in more meaningful ways.
If it is and you do manage to exploit it be sure to write a post about it.
We would be very interested.

# The Mount Propagation Algorithm

When a new mount is attached to a destination - be it a new filesystem or a bind-mount - the `attach_recursive_mnt()` function will be called.
It is also responsible for calling into `propagate_mnt()` to handle mount propagation.

By the time we call into `propagate_mnt()` we know that the destination mount `@dest_mnt` is either a pure shared mount or a shared-slave mount.
This is guaranteed by a check in `attach_recursive_mnt()`.
So `propagate_mnt()` will first propagate the source mount tree to all peers in `@dest_mnt`'s peer group:

```c
for (n = next_peer(dest_mnt); n != dest_mnt; n = next_peer(n)) {
        ret = propagate_one(n);
        if (ret)
               goto out;
}
```

Notice, that the peer propagation loop of `propagate_mnt()` doesn't propagate `@dest_mnt` itself.
Instead, `@dest_mnt` is mounted directly in `attach_recursive_mnt()` after we propagated to the destination propagation tree.

The mount that will be mounted on top of `@dest_mnt` is `@source_mnt`.
This copy was created earlier even before we entered `attach_recursive_mnt()` and doesn't concern us a lot here.

It's just important to notice that when `propagate_mnt()` is called `@source_mnt` will not yet have been mounted on top of `@dest_mnt`.
Thus, `@source_mnt->mnt_parent` will either still point to `@source_mnt` or - in the case `@source_mnt` is moved and thus already attached - still to its former parent.

For each peer `@m` in `@dest_mnt`'s peer group `propagate_one()` will create a new copy of the source mount tree and mount that copy `@child` on `@m` such that `@child->mnt_parent` points to `@m` after `propagate_one()` returns.

`propagate_one()` will stash the last destination propagation node `@m` in `@last_dest` and the last copy it created for the source mount tree in `@last_source`.

Hence, if we call into `propagate_one()` again for the next destination propagation node `@m`, `@last_dest` will point to the previous destination propagation node and `@last_source` will point to the previous copy of the source mount tree and mounted on `@last_dest`.

Each new copy of the source mount tree is created from the previous copy of the source mount tree.
This will become important later.

The peer loop in `propagate_mnt()` is straightforward.
We iterate through the peers copying and updating `@last_source` and `@last_dest` as we go through them and mount each copy of the source mount tree `@child` on a peer `@m` in `@dest_mnt`'s peer group.

After `propagate_mnt()` handles the peers in `@dest_mnt`'s peer group it will propagate the source mount down the propagation tree that `@dest_mnt`'s peer group propagates to:

```c
for (m = next_group(dest_mnt, dest_mnt); m;
                m = next_group(m, dest_mnt)) {
        /* everything in that slave group */
        n = m;
        do {
                ret = propagate_one(n);
                if (ret)
                        goto out;
                n = next_peer(n);
        } while (n != m);
}
```

The `next_group()` helper will recursively walk the destination propagation tree, descending into each propagation group of the propagation tree.

The important part is that it takes care to propagate the source mount tree to all peers in the peer group of a propagation group before it propagates to the slaves to those peers in the propagation group.
In other words, a mount in the source mount propagation tree which will be a master is always created before that mount's slaves.

It is important to remember that propagating the source mount tree to each mount `@m` in the destination propagation tree simply means that we create and mount new copies `@child` of the source mount tree on `@m` such that `@child->mnt_parent` points to `@m`.

Since we know that each node `@m` in the destination propagation tree headed by `@dest_mnt`'s peer group will be overmounted with a copy of the source mount tree and since we know that the propagation properties of each copy of the source mount tree we create and mount at `@m` will mostly mirror the propagation properties of `@m`.
Since we know that each node `@m` in the destination propagation tree headed by `@dest_mnt`'s peer group will be overmounted with a copy of the source mount, and since we know that the propagation properties of each copy of the source mount tree we create and mount at `@m` will mostly mirror the propagation properties of `@m`, we can use that information to create and mount the copies of the source mount that become masters before their slaves.

The easy case is always when `@m` and `@last_dest` are peers in a peer group of a given propagation group.
In that case we know that the new copy `@child` should have the same master as `@last_source`, whose master was determined in a previous call to `propagate_one()`.

The hard case is when we're dealing with an `@m` which is a pure slave mount or a shared-slave mount in a new peer group, as we need to find an appropriate mount in the source mount tree to be the master of `@m`.

For each pure slave or peer group in the destination propagation tree we need to make sure that the master for new copies of `@source_mnt` is a mount from the source mount propagation tree whose parent is in the chain of masters of the parent for the new child mount.
This is a mouthful but as far as we can tell that's the core of it all.

But, if we keep track of the masters in the destination propagation tree `@m` we can use the information to find the correct master for each copy of the source mount tree we create and mount at the slaves in the destination propagation tree `@m`.

Let's walk through the base case as that's still fairly easy to grasp.

If we're dealing with the first slave in the propagation group that `@dest_mnt` is in then we don't yet have marked any masters in the destination propagation tree.

We know the master for the first slave to `@dest_mnt`'s peer group is simply `@dest_mnt`.
So we expect this algorithm to yield a copy of the source mount tree that was mounted on a peer in `@dest_mnt`'s peer group as the master for the copy of the source mount tree we want to mount at the first slave `@m`:

```c
for (n = m; ; n = p) {
        p = n->mnt_master;
        if (p == dest_master || IS_MNT_MARKED(p))
                break;
}
```

For the first slave we walk the destination propagation tree all the way up to a peer in `@dest_mnt`'s peer group.
So, the propagation hierarchy can be walked by walking up the `@m->mnt_master` hierarchy of the destination propagation tree `@m`.
We will ultimately find a peer in `@dest_mnt`'s peer group and thus ultimately `@dest_mnt->mnt_master`.

By the way, here the assumption we listed at the beginning becomes important.
Namely, that peers in a peer group `pg1` that are slaves in another peer group `pg2` appear on the same `->mnt_slave_list`.
So all slaves who are peers in peer group `pg1` point to the same peer in peer group `pg2` via `->mnt_master`.
Otherwise the termination condition in the code above would be wrong and `next_group()` would be broken too.

So the first iteration sets:

```c
n = m;
p = n->mnt_master;
```

such that `@p` now points to a peer or `@dest_mnt` itself.
We walk up one more level since we don't have any marked mounts.
So we end up with:

```c
n = dest_mnt;
p = dest_mnt->mnt_master;
```

If `@dest_mnt`'s peer group is not slave to another peer group then `@p` is now `NULL`.
If `@dest_mnt`'s peer group is a slave to another peer group then `@p` now points to `@dest_mnt->mnt_master`, which is a master outside the propagation tree we're dealing with.

Now we need to figure out the master for the copy of the source mount tree we're about to create and mount on the first slave of `@dest_mnt`'s peer group:

```c
do {
        struct mount *parent = last_source->mnt_parent;
        if (last_source == first_source)
                break;
        done = parent->mnt_master == p;
        if (done && peers(n, parent))
                break;
        last_source = last_source->mnt_master;
} while (!done);
```

We know that `@last_source->mnt_parent` points to `@last_dest` and `@last_dest` is the last peer in `@dest_mnt`'s peer group we propagated to in the peer loop in `propagate_mnt()`.

Consequently, `@last_source` is the last copy we created and mounted on that last peer in `@dest_mnt`'s peer group.
So `@last_source` is the master we want to pick.

We know that `@last_source->mnt_parent->mnt_master` points to `@last_dest->mnt_master`.
We also know that `@last_dest->mnt_master` is either `NULL` or points to a master outside of the destination propagation tree and so does `@p`.
Hence:

```c
done = parent->mnt_master == p;
```

is trivially true in the base condition.

We also know that for the first slave mount of `@dest_mnt`'s peer group, `@last_dest` either points `@dest_mnt` itself because it was initialized to:

```c
last_dest = dest_mnt;
```

at the beginning of `propagate_mnt()` or it will point to a peer of `@dest_mnt` in its peer group.
In both cases it is guaranteed that on the first iteration `@n` and `@parent` are peers (Please note the check for peers here as that's important.):

```c
if (done && peers(n, parent))
        break;
```

So, as we expected, we select `@last_source`, which refers to the last copy of the source mount tree we mounted on the last peer in `@dest_mnt`'s peer group, as the master of the first slave in `@dest_mnt`'s peer group.
The rest is taken care of by `clone_mnt(last_source, ...)`.

At the end of `propagate_mnt()` we now mark `@m->mnt_master` as the first master in the destination propagation tree that is distinct from `@dest_mnt->mnt_master`.
Thus, we mark `@dest_mnt` itself as a master.

By marking `@dest_mnt` or one of it's peers we are able to easily find it again when we later lookup masters for other copies of the source mount tree we mount copies of the source mount tree on slaves `@m` to `@dest_mnt`'s peer group.
This in turn allows us to find the masters we selected for the copies of `@source_mnt`, which are always mounted on masters in the destination propagation tree.

The important part is to realize that the code makes use of the fact that the last copy of the source mount tree stashed in `@last_source` was mounted on top of the previous destination propagation node `@last_dest`.
What this means is that `@last_source` allows us to walk the destination propagation hierarchy the same way each destination propagation node `@m` does.

If we take `@last_source`, which is the copy of `@source_mnt` we have mounted on `@last_dest` in the previous iteration of `propagate_one()`, then we know `@last_source->mnt_parent` points to `@last_dest` but we also know that as we walk through the destination propagation tree that `@last_source->mnt_master` will point to an earlier copy of the source mount tree we mounted one an earlier destination propagation node `@m`.

So `@last_source->mnt_parent` will be our hook into the destination propagation tree and each consecutive `@last_source->mnt_master` will lead us to an earlier propagation node `@m` via `@last_source->mnt_master->mnt_parent`.

Hence, by walking up `@last_source->mnt_master`, each of which is mounted on a node that is a master in the destination propagation tree, we can also walk up the destination propagation hierarchy.

So, for each new destination propagation node `@m` we use the previous copy of `@last_source` and the fact it's mounted on the previous propagation node `@last_dest` via `@last_source->mnt_master->mnt_parent` to determine what the master of the new copy of `@last_source` needs to be.

The goal is to find the _closest_ master that the new copy of the source mount tree we are about to create and mount on a slave `@m` in the destination propagation tree needs to pick.
This means we want to find a suitable master in the propagation group.

As the structure of the source mount propagation tree we create mirrors the propagation structure of the destination propagation
tree we can find `@m`'s closest master - i.e., a marked master - which is a peer in the closest peer group that `@m` receives propagation from.
We store that closest master of `@m` in `@p` as before and record the slave to that master in `@n`

We then search for this master `@p` via `@last_source` by walking up the master hierarchy starting from `@last_source`.

We will try to find the master by walking `@last_source->mnt_master` and by comparing `@last_source->mnt_master->mnt_parent->mnt_master` to `@p`.
If we find `@p` then we can figure out what earlier copy of the source mount tree needs to be the master for the new copy of the source mount tree we're about to create and mount at the current destination propagation node `@m`.

If `@last_source->mnt_master->mnt_parent` and `@n` are peers then we know that the closest master they receive propagation from is `@last_source->mnt_master->mnt_parent->mnt_master`.
If not then the closest immediate peer group that they receive propagation from must be one level higher up.

This builds on the earlier clarification at the beginning that all peers in a peer group which are slaves of other peer groups all point to the same `->mnt_master`, i.e., appear on the same `->mnt_slave_list`, of the closest peer group that they receive propagation from.

# Failing to terminate the algorithm

However, terminating the walk has corner cases.

If the closest marked master for a given destination node `@m` cannot be found by walking up the master hierarchy via `@last_source->mnt_master` then we need to terminate the walk when we encounter `@source_mnt` again.

This isn't an arbitrary termination.
It simply means that the new copy of the source mount tree we're about to create has a copy of the source mount tree we created and mounted on a peer in `@dest_mnt`'s peer group as its master.
So `@source_mnt` is the peer in the closest peer group that the new copy of the source mount tree receives propagation from.

We absolutely have to stop `@source_mnt` because `@last_source->mnt_master` either points outside the propagation hierarchy we're dealing with or it is `NULL` because `@source_mnt` isn't a shared-slave.

So continuing the walk past `@source_mnt` would cause a `NULL` dereference via `@last_source->mnt_master->mnt_parent`.
And so we have to stop the walk when we encounter `@source_mnt` again.

One scenario where this can happen is when we first handled a series of slaves of `@dest_mnt`'s peer group and then encounter peers in a new peer group that is a slave to `@dest_mnt`'s peer group.
We handle them and then we encounter another slave mount to `@dest_mnt` that is a pure slave to `@dest_mnt`'s peer group.
That pure slave will have a peer in `@dest_mnt`'s peer group as its master.
Consequently, the new copy of the source mount tree will need to have `@source_mnt` as it's master.
So we walk the propagation hierarchy all the way up to `@source_mnt` based on `@last_source->mnt_master`.

So terminate on `@source_mnt`, easy peasy.
Except, that the check misses something that the rest of the algorithm already handles.

If `@dest_mnt` has peers in its peer group the peer loop in `propagate_mnt()`:

```c
for (n = next_peer(dest_mnt); n != dest_mnt; n = next_peer(n)) {
        ret = propagate_one(n);
        if (ret)
                goto out;
}
```

will consecutively update `@last_source` with each previous copy of the source mount tree we created and mounted at the previous peer in `@dest_mnt`'s peer group.
So after that loop terminates `@last_source` will point to whatever copy of the source mount tree was created and mounted on the last peer in `@dest_mnt`'s peer group.

Furthermore, if there is even a single additional peer in `@dest_mnt`'s peer group then `@last_source` will __not__ point to `@source_mnt` anymore.
Because, as we mentioned above, `@dest_mnt` isn't even handled in this loop but directly in `attach_recursive_mnt()`.
So it can't even accidently come last in that peer loop.

So the first time we handle a slave mount `@m` of `@dest_mnt`'s peer group the copy of the source mount tree we create will make the __last copy of the source mount tree we created and mounted on the last peer in `@dest_mnt`'s peer group the master of the new copy of the source mount tree we create and mount on the first slave of `@dest_mnt`'s peer group__.

But this means that the termination condition that checks for `@source_mnt` is wrong.
The `@source_mnt` cannot be found anymore by `propagate_one()`.
Instead it will find the last copy of the source mount tree we created and mounted for the last peer of `@dest_mnt`'s peer group again.
And that is a peer of `@source_mnt` not `@source_mnt` itself.

This means, we fail to terminate the loop correctly and ultimately dereference `@last_source->mnt_master->mnt_parent`.
When `@source_mnt`'s peer group isn't slave to another peer group then `@last_source->mnt_master` is `NULL` causing the splat above.

For example, assume `@dest_mnt` is a pure shared mount and has three peers in its peer group:

```
===================================================================================
                                         mount-id   mount-parent-id   peer-group-id
===================================================================================
(@dest_mnt) mnt_master[216]              309        297               shared:216
    \
     (@source_mnt) mnt_master[218]:      609        609               shared:218

(1) mnt_master[216]:                     607        605               shared:216
    \
     (P1) mnt_master[218]:               624        607               shared:218

(2) mnt_master[216]:                     576        574               shared:216
    \
     (P2) mnt_master[218]:               625        576               shared:218

(3) mnt_master[216]:                     545        543               shared:216
    \
     (P3) mnt_master[218]:               626        545               shared:218
```

After this sequence has been processed `@last_source` will point to `(P3)`, the copy generated for the third peer in `@dest_mnt`'s peer group we handled.
So the copy of the source mount tree `(P4)` we create and mount on the first slave of `@dest_mnt`'s peer group:

```
===================================================================================
                                         mount-id   mount-parent-id   peer-group-id
===================================================================================
    mnt_master[216]                      309        297               shared:216
   /
  /
(S0) mnt_slave                           483        481               master:216
  \
   \    (P3) mnt_master[218]             626        545               shared:218
    \  /
     \/
    (P4) mnt_slave                       627        483               master:218
```

will pick the last copy of the source mount tree `(P3)` as master, not `(@source_mnt)`.

When walking the propagation hierarchy via `@last_source`'s master hierarchy we encounter `(P3)` but not `(@source_mnt)`.

We can fix this in multiple ways:

(1) By setting `@last_source` to `@source_mnt` after we processed the peers in `@dest_mnt`'s peer group right after the peer loop in `propagate_mnt()`.
    This guarantees that we really alwways find `@source_mnt` itself.

(2) By changing the termination condition that relies on finding exactly `@source_mnt` to finding a peer of `@source_mnt`.

(3) By only moving `@last_source` when we actually venture into a new peer group or some clever variant thereof.

The first two options are minimally invasive and what we want as a fix.
The third option is more intrusive but something we'd like to explore in the near future.

# Conclusion

This is an example of a very clever but __worringly__ underdocumented algorithm.
Since there isn't a single detailed comment to be found in the code it has been a giant pain to understand and work through this bug.
A bug like this is very difficult to fix without a detailed understanding of what's happening.
Let's not talk about the amount of time that was sunk into fixing this.

Oh, and as predicted the actual [fix](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=11933cf1d91d57da9e5c53822a540bbdc2656c16) was trivial.
