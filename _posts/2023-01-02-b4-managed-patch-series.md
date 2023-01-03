---
layout: post
title: Managing a kernel patch series with b4
---

This is a (live-[?])blog about managing a patch series solely with `b4`.
It's "live" as I'm writing this down while fumbling my way through this adventure.
No corrections other than grammer and spelling (but no guarantees for the correctness of either).

Ok, I have a `xfstests` patch to send.
The patch has been written and tested.
I've already created a branch `fstests.setgid.v6.2` based on `xfstests`'s `for-next` branch.

So, first step locate the subcommand that will allow me to do something with that patch/branch.

```
b4 --help
```

That shows a subcommand `prep` which looks like what I would want:

```
> b4 prep --help
usage: b4 prep [-h] [-c | -p OUTPUT_DIR | --edit-cover | --show-revision | --force-revision N | --compare-to vN | --manual-reroll COVER_MSGID | --set-prefixes PREFIX [PREFIX ...] |
               --show-info] [-n NEW_SERIES_NAME] [-f FORK_POINT] [-F MSGID] [-e ENROLL_BASE]

options:
  -h, --help            show this help message and exit
  -c, --auto-to-cc      Automatically populate cover letter trailers with To and Cc addresses
  -p OUTPUT_DIR, --format-patch OUTPUT_DIR
                        Output prep-tracked commits as patches
  --edit-cover          Edit the cover letter in your defined $EDITOR (or core.editor)
  --show-revision       Show current series revision number
  --force-revision N    Force revision to be this number instead
  --compare-to vN       Display a range-diff to previously sent revision N
  --manual-reroll COVER_MSGID
                        Mark current revision as sent and reroll (requires cover letter msgid)
  --set-prefixes PREFIX [PREFIX ...]
                        Extra prefixes to add to [PATCH] (e.g.: RFC mydrv)
  --show-info           Show current series info in a column-parseable format

Create new branch:
  Create a new branch for working on patch series

  -n NEW_SERIES_NAME, --new NEW_SERIES_NAME
                        Create a new branch for working on a patch series
  -f FORK_POINT, --fork-point FORK_POINT
                        When creating a new branch, use this fork point instead of HEAD
  -F MSGID, --from-thread MSGID
                        When creating a new branch, use this thread

Enroll existing branch:
  Enroll existing branch for prep work

  -e ENROLL_BASE, --enroll ENROLL_BASE
                        Enroll current branch, using the passed tag, branch, or commit as fork base

```

Ok, so looks like I first need to enroll that branch.
It wants a base so I'm going to use `HEAD~1` as I'm basing this on the `for-next` branch of `xfstests`:

```
> b4 prep -e HEAD~1
Will track 1 commits
Created the default cover letter, you can edit with --edit-cover.
```

So that worked.
First question I have is whether I can exmatriculate a branch?

Ah, neat it seems that the cover letter is kept as an empty commit before the actual commit:

```
commit 416776f9204e73dfc6900c5e41a7aa47e869203a
Author:     Christian Brauner <brauner@kernel.org>
AuthorDate: Tue Jan 3 15:43:06 2023 +0100
Commit:     Christian Brauner (Microsoft) <brauner@kernel.org>
CommitDate: Tue Jan 3 15:43:06 2023 +0100

    EDITME: cover title for fstests.setgid.v6.2

    # Lines starting with # will be removed from the cover letter. You can use
    # them to add notes or reminders to yourself.

    EDITME: describe the purpose of this series. The information you put here
    will be used by the project maintainer to make a decision whether your
    patches should be reviewed, and in what priority order. Please be very
    detailed and link to any relevant discussions or sites that the maintainer
    can review to better understand your proposed changes. If you only have a
    single patch in your series, the contents of the cover letter will be
    appended to the "under-the-cut" portion of the patch.

    # You can add trailers to the cover letter. Any email addresses found in
    # these trailers will be added to the addresses specified/generated during
    # the b4 send stage. You can also run "b4 prep --auto-to-cc" to auto-populate
    # the To: and Cc: trailers based on the code being modified.

    Signed-off-by: Christian Brauner (Microsoft) <brauner@kernel.org>

    --- b4-submit-tracking ---
    # This section is used internally by b4 prep for tracking purposes.
    {
      "series": {
        "revision": 1,
        "change-id": "20230103-fstests-setgid-v6-2-4ce5852d11e2",
        "base-branch": null,
        "prefixes": []
      }
    }
```

My solution so far had been to keep it as a branch description.
I don't have a strong opinion about this though I'm not sure I like the empty commit thing.
But the motivation most likely is that branch descriptions aren't kept/synced across remotes whereas an empty commit is.

In any case, I'm not concinved that my single patch needs a cover letter so I need to figure out how to get rid of it.
Given that I don't see a direct command to achieve this the first thing that comes to mind is to make the empty commit's commit message empty.
Let's try that:

```
> b4 prep --edit-cover
New cover letter blank, leaving current one unchanged.
```

Let's put a comment under there?

```
commit 8c6a27fc5b793a4a23bb9d457662ed821b5431df
Author:     Christian Brauner <brauner@kernel.org>
AuthorDate: Tue Jan 3 15:43:06 2023 +0100
Commit:     Christian Brauner (Microsoft) <brauner@kernel.org>
CommitDate: Tue Jan 3 15:43:06 2023 +0100

    # I don't need a cover letter.

    --- b4-submit-tracking ---
    # This section is used internally by b4 prep for tracking purposes.
    {
      "series": {
        "revision": 1,
        "change-id": "20230103-fstests-setgid-v6-2-4ce5852d11e2",
        "base-branch": null,
        "prefixes": []
      }
    }
```

It seems that for the next step I might need to use `b4 send`. It provides
`--dry-run` and `--reflect` where the former option is pretty self-explanatory
and the latter option allow to send the patch series to myself.

Let's try that:

```
> b4 send --reflect
Converted the branch to 1 messages
---
To: "Christian Brauner (Microsoft)" <brauner@kernel.org>
---
  [PATCH] generic: update setgid tests
    +Cc: Amir Goldstein <amir73il@gmail.com>
         Zorro Lang <zlang@redhat.com>
---
Ready to:
  - send the above messages to just Christian Brauner <brauner@kernel.org> (REFLECT MODE)
  - with envelope-from: Christian Brauner <brauner@kernel.org>
  - via SMTP server mail.kernel.org

REFLECT MODE:
    The To: and Cc: headers will be fully populated, but the only
    address given to the mail server for actual delivery will be
    Christian Brauner <brauner@kernel.org>

    Addresses in To: and Cc: headers will NOT receive this series.

Press Enter to proceed or Ctrl-C to abort
```

That looks good.
Excellent that it points out that it will only be sent to me and no one else.
I like this a lot!

Ah, that was skinny love.

```
Press Enter to proceed or Ctrl-C to abort
Connecting to mail.kernel.org:587
Traceback (most recent call last):
  File "/home/brauner/src/git/b4/b4/command.py", line 376, in <module>
    cmd()
  File "/home/brauner/src/git/b4/b4/command.py", line 359, in cmd
    cmdargs.func(cmdargs)
  File "/home/brauner/src/git/b4/b4/command.py", line 86, in cmd_send
    b4.ez.cmd_send(cmdargs)
  File "/home/brauner/src/git/b4/b4/ez.py", line 1523, in cmd_send
    sent = b4.send_mail(smtp, send_msgs, fromaddr=fromaddr, patatt_sign=sign,
  File "/home/brauner/src/git/b4/b4/__init__.py", line 3257, in send_mail
    bdata = patatt.rfc2822_sign(bdata)
AttributeError: module 'patatt' has no attribute 'rfc2822_sign'
```

Chances are that my `patatt` version is too old?
Ok, let's try `pip3 install --upgrade patatt` and try this again:

```
> b4 send --reflect
Converted the branch to 1 messages
---
To: "Christian Brauner (Microsoft)" <brauner@kernel.org>
---
  [PATCH] generic: update setgid tests
    +Cc: Amir Goldstein <amir73il@gmail.com>
         Zorro Lang <zlang@redhat.com>
---
Ready to:
  - send the above messages to just Christian Brauner <brauner@kernel.org> (REFLECT MODE)
  - with envelope-from: Christian Brauner <brauner@kernel.org>
  - via SMTP server mail.kernel.org

REFLECT MODE:
    The To: and Cc: headers will be fully populated, but the only
    address given to the mail server for actual delivery will be
    Christian Brauner <brauner@kernel.org>

    Addresses in To: and Cc: headers will NOT receive this series.

Press Enter to proceed or Ctrl-C to abort
Connecting to mail.kernel.org:587
---
  [PATCH] generic: update setgid tests
---
Reflected 1 messages
```

There we go!
Let's inspect whether the patch looks sane.

I would really appreciate if I didn't have to carry a pointless empty commit message with the commend `# I don't need a cover letter.`:

```
commit 8c6a27fc5b793a4a23bb9d457662ed821b5431df
Author:     Christian Brauner <brauner@kernel.org>
AuthorDate: Tue Jan 3 15:43:06 2023 +0100
Commit:     Christian Brauner (Microsoft) <brauner@kernel.org>
CommitDate: Tue Jan 3 15:43:06 2023 +0100

    # I don't need a cover letter.

    --- b4-submit-tracking ---
    # This section is used internally by b4 prep for tracking purposes.
    {
      "series": {
        "revision": 1,
        "change-id": "20230103-fstests-setgid-v6-2-4ce5852d11e2",
        "base-branch": null,
        "prefixes": []
      }
    }
```

Seems like that could be improved by at least allowing no content other than the `b4-submit-tracking` content in there.

Another thought, while `b4 send` allows to specify recipients via `--to` and `--cc` it would be neat if one could store additional recipients alongside each patch outside of trailers.
Currently I do this by tagging each commit I generate via `git format-patch` based on `git notes` which are then moved before the `SUBJECT` line.
For example:

```
From 8575998dc5ac659f9d893220372cabbbb65c323d Mon Sep 17 00:00:00 2001
From: Christian Brauner <a@b>
Date: Fri, 23 Sep 2022 10:29:39 +0200
To: A <a@a>
Cc: B <b@b> 
Cc: C <c@c> 
Cc: D <d@d> 
Cc: E <e@e> 
Subject: [PATCH v5 02/30] some: commmimt
```

It be excellent if that were possible.
For the single patch I can do with just using the `--to` and `--cc` flags to `b4 send`.
So let's try this but better safe than sorry and also let's take the chance to try `--dry-run`:

```
> b4 send --cc="Amir Goldstein <amir73il@gmail.com>" --cc="Zorro Lang <zlang@redhat.com>" --to="<fstests@vger.kernel.org>" --dry-run
Converted the branch to 1 messages
    --- DRYRUN: message follows ---
    | From: Christian Brauner <brauner@kernel.org>
    | Date: Tue, 03 Jan 2023 16:26:26 +0100
    | Subject: [PATCH] generic: update setgid tests
    | MIME-Version: 1.0
    | Content-Type: text/plain; charset="utf-8"
    | Content-Transfer-Encoding: 7bit
    | Message-Id: <20230103-fstests-setgid-v6-2-v1-1-cbdbeef2411a@kernel.org>
    | X-B4-Tracking: v=1; b=H4sIACJJtGMC/x2NwQqDMBAFf0X23IVkraX0V0oPMXnqgqQlG6Ug/
    |  ntDjzMwzEGGojB6dAcV7Gr6zg38paO4hDyDNTUmcdI773qerMKqsaHOmni/sfA1YrgPkryHUCvH
    |  YOCxhByX1uZtXZv8FEz6/a+er/P8AY8ALQl6AAAA
    | To: fstests@vger.kernel.org
    | Cc: "Christian Brauner (Microsoft)" <brauner@kernel.org>, Zorro Lang <zlang@redhat.com>,
    |  Amir Goldstein <amir73il@gmail.com>
    | X-Mailer: b4 0.12-dev-214b3
    | X-Developer-Signature: v=1; a=openpgp-sha256; l=11235; i=brauner@kernel.org;
    |  h=from:subject:message-id; bh=Oi7CQzED3JPkJt0k+GEUAJ3KgV2hlR5H9UVIMTL7D54=;
    |  b=owGbwMvMwCU28Zj0gdSKO4sYT6slMSRv8VRqrb5e/2BddGniFuONH66XL6oU33feadWtOWfnTmMR
    |  M157sqOUhUGMi0FWTJHFod0kXG45T8Vmo0wNmDmsTCBDGLg4BWAi8+YzMhz+fd3m3CfdR3/fvFxy9t
    |  Xj36frBTq/Vgvy/isXDvRg7Gdh+J8ZvXPBu2W8mwpPd8ZbXZn3alVEZq9XiPk2YSt5yZVftLgB
    | X-Developer-Key: i=brauner@kernel.org; a=openpgp;
    |  fpr=4880B8C9BD0E5106FC070F4F7B3C391EFEA93624
    |

[I'm snipping the actual content as this is not of interest here.]

    | ---
    | base-commit: fbd489798b31e32f0eaefcd754326a06aa5b166f
    | change-id: 20230103-fstests-setgid-v6-2-4ce5852d11e2
    |
    | Best regards,
    | --
    | Christian Brauner (Microsoft) <brauner@kernel.org>
    --- DRYRUN: message ends ---
---
DRYRUN: Would have sent 1 messages
```

So let's try this for real:

```
> b4 send --cc="Amir Goldstein <amir73il@gmail.com>" --cc="Zorro Lang <zlang@redhat.com>" --to="<fstests@vger.kernel.org>"
Converted the branch to 1 messages
---
To: fstests@vger.kernel.org
Cc: "Christian Brauner (Microsoft)" <brauner@kernel.org>
    Zorro Lang <zlang@redhat.com>
---
  [PATCH] generic: update setgid tests
    +Cc: Amir Goldstein <amir73il@gmail.com>
---
Ready to:
  - send the above messages to actual listed recipients
  - with envelope-from: Christian Brauner <brauner@kernel.org>
  - via SMTP server mail.kernel.org
  - tag and reroll the series to the next revision

Press Enter to proceed or Ctrl-C to abort
Connecting to mail.kernel.org:587
---
  [PATCH] generic: update setgid tests
---
Sent 1 messages
Tagging sent/fstests.setgid.v6.2-v1
Recording series message-id in cover letter tracking
Created new revision v2
Updating cover letter with templated changelog entries.
Invoking git-filter-repo to update the cover letter.
New history written in 0.02 seconds...
Completely finished after 0.05 seconds.
```

Ok, let's check my own mail whether this looks good.
And yes, it does:

```
Date: Tue, 03 Jan 2023 16:28:20 +0100
From: Christian Brauner <brauner@kernel.org>
To: fstests@vger.kernel.org
Cc: "Christian Brauner (Microsoft)" <brauner@kernel.org>, Zorro Lang <zlang@redhat.com>, Amir Goldstein <amir73il@gmail.com>
Subject: [PATCH] generic: update setgid tests
X-Date: Tue, 03 Jan 2023 16:28:20 +0100
X-URI: https://lore.kernel.org/fstests/20230103-fstests-setgid-v6-2-v1-1-b8972c303ebe@kernel.org
```
