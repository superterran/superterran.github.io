---
title: "the host was oom but docker said it wasn't"
date: 2026-07-06
draft: false
categories: ["homelab"]
tags: ["docker", "truenas", "oom", "gotcha"]
summary: "OOMKilled: false in docker inspect doesn't mean OOM didn't happen. The host's global OOM killer doesn't set that flag — dmesg does."
---

The gateway started crash-looping. Container logs showed nothing useful. Restarted several times, but `docker inspect` on it came back with `"OOMKilled": false` — so OOM seemed ruled out.

It wasn't.

```
$ dmesg | grep -i oom
[...] Out of memory: Killed process <pid> (openclaw-gateway)
[...] Out of memory: Killed process <pid> (openclaw-gateway)
```

When the kernel's global OOM killer takes out a container, Docker records the restart but doesn't set `OOMKilled: true`. That flag only flips when the container hits its *own* cgroup memory limit. Without a `mem_limit` in Compose, there's no cgroup ceiling, so the flag never triggers — even while the host is completely out of RAM.

The box had 11GB RAM and zero swap. Running the full app stack left almost nothing for OS headroom. The gateway (sitting at about 1GB RSS) kept getting picked as the largest killable process.

Fix: stopped the burst-only containers — torrent client, a few media stack apps, a leaky companion proxy — and added 16GB raw swap on a spare SATA disk.

One thing specific to TrueNAS: don't put swap on ZFS. ZFS needs memory to handle I/O, so a ZFS zvol as swap can deadlock under memory pressure — the system needs memory to page in, which it doesn't have. A dedicated raw disk partition doesn't have that coupling.

After parked containers plus swap: gateway stable, available RAM went from under 500MB to about 2.4GB.
