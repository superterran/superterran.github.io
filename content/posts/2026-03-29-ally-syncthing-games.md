---
title: "provisioning the rog ally into the shared game library"
date: 2026-03-29
draft: false
categories: ["homelab"]
tags: ["gaming", "syncthing", "rog-ally", "steam", "bazzite", "ssh"]
summary: "Adding the ROG Ally as a Syncthing peer. Battery-friendly settings, SSH remote provisioning, five games migrated from SD card to internal NVMe."
---

With the shared-games Syncthing folder running on bazzite-desktop, the next step was adding the ROG Ally. The Ally runs Bazzite — same OS family, same tooling — so provisioning it remotely via SSH is straightforward.

## Battery-friendly Syncthing config

The Ally is battery-powered. Default Syncthing settings assume a plugged-in machine: global relay servers, wide discovery scope, frequent rescans. On the Ally:

- Global relays disabled (LAN-only sync)
- Only local discovery enabled
- Rescan interval: 10 minutes (vs. default 60 seconds)
- Filesystem watcher: enabled (so changes trigger sync even with the long rescan interval)
- Reconnect interval: 120 seconds

The Ally and desktop only need to sync when on the same LAN, so there's no reason to route through relay servers.

## SSH and mDNS

Added the Ally to `~/.ssh/config` using its LAN IP directly. `ally.local` resolves fine via `avahi-resolve` in a terminal but doesn't reliably work as the `HostName` in `~/.ssh/config` from a bash script context. IP is stable enough for home LAN.

## Games migrated

Five games moved from the Ally's SD card to `shared-games` on the internal NVMe:

| Game | Type | Save sync |
|------|------|-----------|
| GhostShip (SM64 PC port) | AppImage | In game dir |
| Ship of Harkinian (OoT PC port) | AppImage | In game dir |
| AM2R | Windows exe via Proton | Symlink from Proton prefix |
| Link's Awakening DX HD | Windows exe via Proton | SaveFiles/ in game dir |
| Super Mario Bros. Remastered | Native Linux (Godot) | Symlink to `~/.local/share/SMB1R/saves/` |

The Proton-prefix games need a symlink so the Proton prefix points into the shared-games saves directory rather than keeping saves inside the prefix where they'd stay machine-local.

## SM64 Remastered save symlink pattern

SMB1R is a Godot game that writes saves to `~/.local/share/SMB1R/saves/`. The game doesn't have a config option to change this path. The workaround: put the actual saves directory inside `shared-games/smb1r/saves/`, then symlink `~/.local/share/SMB1R/saves/` to point there. Syncthing syncs the shared-games folder; the game sees the path it expects.

The same pattern works for any game with a hardcoded save path.

## Ally Syncthing device path

The Ally's shared-games folder lives at `/home/bazzite/shared-games` — not `/var/mnt/fast/shared-games` like the desktop. Different paths, same Syncthing folder. This matters for Steam: shortcut `appid` is a CRC32 of the exe path plus the game name. Different paths mean different appids on each machine. Artwork lookup by appid breaks cross-device. The fleet setup session addressed this by switching to slug-keyed artwork caching.

## Provisioning script

`ally-setup.sh` automates the full remote setup: SSH key copy, Syncthing install, service enable, folder config via REST API, game setup script run. One script, one command from the desktop, Ally fully provisioned.
