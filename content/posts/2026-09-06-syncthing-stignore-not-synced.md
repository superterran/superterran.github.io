---
title: "needFiles: 0 on both sides, still out of sync"
date: 2026-09-06
draft: false
categories: ["homelab"]
tags: ["syncthing", "linux", "symlinks"]
summary: "A shared Syncthing folder read healthy on both machines while quietly missing a whole directory on one side, because the file that would have caught it isn't itself synced."
---

Two machines, one Syncthing folder, `needFiles: 0` on both sides. That's the number you check when you want to know if a sync is caught up. It said yes.

It also said 3028 local items on one machine and 3128 on the other.

The gap was a whole game directory — 99 files, a few hundred megabytes — that one side was quietly ignoring and the other was happily replicating in. `.stignore` is per-folder, per-machine, and Syncthing does not sync it. Copy the folder to a second device without also copying `.stignore` by hand, and that device syncs in everything the first device was told to skip. Nothing fails. Nothing errors. `needFiles: 0` just means "I have everything I've been told to want," and the two machines had been told to want different things.

The fix was one file, copied over. The more interesting part was how long it stayed invisible — local item counts are the only signal that catches it, and nobody watches those day to day.

Same pass turned up a smaller version of the same shape. A handful of ROM symlinks inside the folder held absolute paths from the machine that created them. Syncthing replicates a symlink by copying the target string, not by resolving it, so a link built against one machine's mount point lands on the other machine pointing at a path that doesn't exist there — even though the file it should resolve to synced in just fine. Relative paths fixed it for good.

Both bugs are the same thing wearing different clothes: something that isn't itself content, sitting alongside the content, not participating in the sync contract, and quietly diverging. The health check only checks the contract.
