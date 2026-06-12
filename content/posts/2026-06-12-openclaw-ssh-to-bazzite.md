---
title: "ssh inside a capdrop=all container"
date: 2026-05-23
draft: false
categories: ["homelab"]
tags: ["openclaw", "truenas", "closet", "ssh", "docker", "bazzite"]
summary: "Getting SSH working inside the OpenClaw catalog app on TrueNAS SCALE: two silent blockers, one binary bundle, one passwd seed."
---

The OpenClaw container on closet needed SSH access to bazzite so agents running inside it could use the GPU workstation as a dev box. Getting there involved two blockers that failed in completely different ways.

## The container

OpenClaw runs as a TrueNAS SCALE catalog app. These containers run with `CapDrop=ALL` and TrueNAS overrides the image's `USER` directive — the image says `node:1000`, the container runs as `apps:568`.

The first instinct was `apt-get install openssh-client`. That failed, but not loudly. `apt-get` runs the `_apt` drop-privilege step via `setgid`, and `CapDrop=ALL` removes `CAP_SETGID`. The install appeared to proceed and then silently produced nothing. No binary, no error worth noticing.

Forking the catalog image to bake SSH in was the obvious next option. I ruled it out — maintenance burden every upgrade cycle isn't worth it for one tool.

## The binary bundle

Closet's host OS is Debian 12. The container is also Debian 12. They're binary-compatible. The SSH binary on the host works inside the container.

The approach: copy `/usr/bin/ssh` from the closet host filesystem into a persistent bind-mounted directory inside the container, then symlink it onto PATH.

```bash
# on the closet host
cp /usr/bin/ssh /path/to/persistent-mount/ssh-bundle/bin/ssh
```

No library bundling needed. Everything `ssh` links against (`libcrypto`, `libz`, `libc`) is already present in `/lib/x86_64-linux-gnu/` inside the container — Debian base image.

The persistent bind mount survives catalog upgrades. The PATH symlinks do not — those live in the container's writable layer and get wiped on recreation. They go in the post-upgrade runbook.

## The passwd problem

With the binary in place, `ssh -V` died immediately:

```
No user exists for uid 568
```

OpenSSH calls `getpwuid(getuid())` at startup. If it returns NULL, the client aborts before doing anything. The container UID is 568 (TrueNAS override), but `/etc/passwd` only has an entry for `node:1000`. There's no `568` there.

The fix is a passwd seed — add a `node:568` line to `/etc/passwd` inside the container:

```
node:x:568:568::/home/node:/bin/bash
```

This entry lives in the container's writable layer. It works immediately, no restart. Like the PATH symlinks, it needs to be re-seeded after container recreation — same runbook step.

## The key

Ed25519 keypair generated inside the container, public key installed on bazzite with a source-IP restriction and `no-agent-forwarding,no-X11-forwarding`. Minimal blast radius if the key were ever extracted from the container.

After that: `ssh bazzite hostname` from inside the container returned `desktop`. Done.

## What doesn't survive upgrades

Two things need re-running after a TrueNAS catalog app upgrade or container recreation:

1. Re-link `ssh` onto PATH from the persistent bundle
2. Re-seed `/etc/passwd` with the UID 568 entry

Both are documented in the ops runbook for this install. Neither is complicated, just easy to forget.
