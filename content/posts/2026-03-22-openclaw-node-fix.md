---
title: "podman overlay broken on bazzite: migrating the openclaw node to docker"
date: 2026-03-22
draft: false
categories: ["homelab"]
tags: ["bazzite", "openclaw", "podman", "docker", "systemd"]
summary: "The OpenClaw local node was in a restart loop. Podman rootless overlay storage was broken on this host; Docker wasn't. Swap the runtime, add persistent identity storage, re-pair."
---

The OpenClaw node service on bazzite was stuck in a restart loop. Journal showed the crash: Podman rootless overlay storage failing with "operation not permitted." Podman rootless mode needs user namespace support and specific kernel settings — on this Bazzite install they weren't cooperating. Docker on the same host worked fine.

## The fix: swap the runtime

The original unit was a Podman Quadlet (`.container` file in `~/.config/containers/systemd/`). Dropped that and replaced it with a plain systemd user service using `docker run`.

Podman and Docker are largely interchangeable for simple container runs. The Quadlet syntax is tidier, but not when the underlying storage driver is broken.

## Node identity persistence

First attempt at the Docker-based unit came up and immediately required re-pairing. The node was generating new cryptographic identity on every start because `~/.openclaw` inside the container wasn't persisted.

Fix: add a bind mount from a directory on the host to the node's home path inside the container. On next start, the node read its existing `node.json` and `identity/device.json`, reconnected with its established identity, and pairing wasn't required.

This is easy to forget with agent/node containers: if the identity lives inside the container's ephemeral layer, every restart looks like a brand new device to the gateway.

## Re-pairing

Once the node came up with a stable identity, it still needed its pairing approved on the gateway side — a one-time step since the old Podman-based device entry was stale. Ran `openclaw devices approve --latest` from the OpenClaw gateway container on closet. After that: both tunnel and node services active, no crash loop.

## What the remote side looked like the whole time

The remote OpenClaw gateway on closet was healthy throughout. The failure was entirely local — bazzite's container runtime. The gateway had been accumulating stale "pairing required" entries from the restart loop generating new identities. Worth cleaning those up after the fix.
