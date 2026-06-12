---
title: "slug-keyed artwork cache and fleet provisioning for shared games"
date: 2026-04-04
draft: false
categories: ["homelab"]
tags: ["gaming", "syncthing", "steam", "rog-ally", "bazzite", "steamgriddb"]
summary: "Artwork keyed by slug (not appid) so it syncs across machines. sync-peers.sh provisions the whole fleet from one command. Scripts pushed to GitHub."
---

Two things were missing after adding the Ally to the shared game library: artwork was being downloaded per-machine because it was keyed by Steam appid (which differs between devices when exe paths differ), and provisioning a new device still required running setup separately on each machine. Both got fixed.

## Why appid-keyed artwork breaks across machines

Steam generates the appid for non-Steam shortcuts by CRC32-hashing the exe path and app name. Since the desktop mounts shared-games at `/var/mnt/fast/shared-games/` and the Ally at `/home/bazzite/shared-games/`, the exe paths differ, the appids differ, and artwork downloaded on one machine doesn't match what the other machine looks for.

The fix: key the artwork cache by game slug (a stable string I control), not by appid. Store it in `_artwork/<slug>/` inside the Syncthing folder. When `setup-steam.sh` runs on any device, it:

1. Checks if artwork exists in the local cache for this slug
2. If not, downloads from SteamGridDB (requires API key)
3. Copies from the slug-keyed cache to Steam's `grid/` directory using the local appid

Syncthing propagates the `_artwork/` directory. Second device runs setup, finds artwork already cached, copies it without hitting the API. One download serves the whole fleet forever.

## sync-peers.sh

`sync-peers.sh` runs the setup locally first, then SSHes to each fleet peer and runs it there too. It detects the current hostname to skip itself:

```bash
for peer in desktop ally; do
  [ "$peer" = "$(hostname -s)" ] && continue
  ssh "$peer" "STEAMGRIDDB_API_KEY=$STEAMGRIDDB_API_KEY shared-games/_setup/setup-steam.sh"
done
```

One command from either machine provisions the whole fleet. The API key is forwarded over SSH when present; peers without a key still get artwork from the Syncthing cache.

## Bidirectional by default

`sync-peers.sh` is designed to run from either device — not just the desktop. If the Ally adds a new game, it can run provisioning and the desktop gets the shortcut on the next sync. The script doesn't assume a master/replica relationship.

## Scripts on GitHub

`_setup/` is a git repo tracking `superterran/shared-games-setup`. Syncthing propagates the scripts to all devices; git tracks their history. No separate sync mechanism needed — the scripts live in the thing they manage.

## Current state

Desktop and Ally: five games fully configured with Steam shortcuts and artwork. `mario64` is present in the folder structure but the binary needs to be built from source — the script skips it gracefully when no binary is found. SNES ROM hacks need EmuDeck per-device, not automated.
