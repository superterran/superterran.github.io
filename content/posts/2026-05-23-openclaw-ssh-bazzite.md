---
title: "ssh in a capdrop=all container: getting openclaw talking to bazzite"
date: 2026-05-23
draft: false
categories: ["homelab"]
tags: ["openclaw", "ssh", "truenas", "bazzite", "closet", "containers"]
summary: "TrueNAS catalog apps drop every Linux capability, so apt-get fails silently. Here's how to get a working SSH client into a locked-down container using a binary bundle from the host."
---

The OpenClaw container on closet needed to be able to SSH into bazzite-desktop so agents could run dev-box commands remotely. Simple enough — except this container runs as a TrueNAS catalog app, and TrueNAS sets `CapDrop=ALL` on every catalog container.

## Why apt-get fails silently

First instinct: `apt-get install openssh-client`. It ran, seemed to complete, and produced no binary. No error either.

The problem is `CapDrop=ALL` strips the `setgid` capability that the `_apt` user-dropping mechanism relies on during package installation. The install appears to succeed but nothing lands. No obvious error, just a missing binary at the end.

Forking the catalog image to add SSH during build was the next thought, but that means maintaining a custom image through every OpenClaw release. Not worth it.

## Binary bundle from the host

The closet host and the OpenClaw container are both Debian 12, which means the host's `/usr/bin/ssh` is binary-compatible with the container's environment. The solution: copy the SSH binary (and its dependencies) from the closet host into a directory that's bind-mounted into the container as a persistent volume.

```bash
# On the closet host
cp /usr/bin/ssh /mnt/ix-openclaw-data/ssh-bundle/bin/
ldd /usr/bin/ssh  # all libs were already in the container
```

All the required shared libraries were already present in the container's `/lib/x86_64-linux-gnu/`. No library bundling needed — just the binary.

Inside the container, symlinks in `/home/node/.openclaw/.openclaw/bin/` point into the bundle directory. That path is on `$PATH`. The bundle directory survives catalog upgrades because it sits on a persistent bind mount. The symlinks don't — they live in the writable container layer and need to be re-linked after any catalog upgrade or container recreation.

## The uid mismatch

Getting the binary in place wasn't the end of it. Running `ssh -V` returned:

```
No user exists for uid 568
```

TrueNAS overrides the image's `USER` directive, running the container as `568:568` (the TrueNAS `apps` service account). The container's `/etc/passwd` has `node:1000` — uid 568 isn't listed at all. SSH calls `getpwuid(getuid())` early in startup and aborts if it gets NULL back. It needs a valid passwd entry for whatever uid the process is running as.

Fix: seed `/etc/passwd` with a line mapping 568 to the node home:

```
node:x:568:568::/home/node:/bin/bash
```

This line lives in the container's writable layer. It does not survive container recreation — same caveat as the symlinks. Both steps go in the upgrade runbook.

## Keypair setup

With SSH operational, the rest is standard:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/openclaw_to_bazzite -N ""
```

On bazzite, the public key goes in `~/.ssh/authorized_keys` with a `from=` restriction limiting connections to closet's LAN IP:

```
from="10.0.0.75",no-agent-forwarding,no-X11-forwarding ssh-ed25519 AAAA...
```

The `from=` restriction means the key is useless even if it leaks — it only works from inside the LAN, from that specific host IP.

## What survives an upgrade

The bundle at `/mnt/ix-openclaw-data/ssh-bundle/bin/ssh` survives. The symlinks and the passwd seed don't. The runbook now has a step 11 that re-applies both after any catalog upgrade or container recreation. Without it, the next upgrade silently breaks SSH again — the same kind of invisible failure that broke `apt-get` in the first place.
