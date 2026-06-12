---
title: "shared game library with syncthing and steam shortcuts"
date: 2026-03-28
draft: false
categories: ["homelab"]
tags: ["gaming", "syncthing", "steam", "bazzite", "python"]
summary: "A Syncthing folder for games shared between the desktop and the ROG Ally. Non-Steam shortcuts via the VDF library. One tricky int32 sign issue."
---

The goal: a game library that lives on a fast NVMe drive, syncs between bazzite-desktop and the ROG Ally, and shows up in Steam with proper artwork. Both machines can access the same saves. A setup script handles adding shortcuts so the process is repeatable when a new game gets added.

## Structure

```
/var/mnt/fast/shared-games/
  ghostship/         # SM64 GhostShip PC port
    ghostship.AppImage
    saves/
    rom/             # stignored — ROMs don't sync
  _setup/
    setup-steam.sh   # adds shortcuts + downloads artwork
    update-games.sh  # checks for new releases
  .stignore          # ROM dirs excluded
```

ROMs are copyright-held and don't sync. Games that need them generate processed asset files on first launch (`.o2r`, `.otr`); those files do sync, so only one machine needs to do the generation step.

## Adding non-Steam shortcuts via VDF

Steam stores non-Steam shortcuts in a binary VDF file (`~/.local/share/Steam/userdata/<uid>/config/shortcuts.vdf`). The `vdf` Python library reads and writes this format. Running it via `uv run --with vdf` means no pip install required on any machine — uv handles the temporary environment.

One gotcha: the VDF binary format stores the shortcut `appid` as a signed int32. Steam generates these by CRC32-hashing the exe path + app name. CRC32 values with the high bit set are negative when interpreted as signed int32 — passing them as Python's unsigned int raises a `struct.error`. Fix:

```python
import ctypes
appid = ctypes.c_int32(unsigned_crc32_value).value
```

## Syncthing setup

Syncthing 2.x stores config at `~/.local/state/syncthing/config.xml` — not `~/.config/syncthing/` as older docs suggest. The service can be added via REST API after install, which is what the setup script does rather than manually editing XML.

## SteamGridDB artwork

The setup script downloads capsule, wide, hero, and logo art from SteamGridDB when an API key is present in `~/.env`. Without the key it skips artwork gracefully. Artwork handling became more interesting once the Ally was added — see the fleet setup post.

## Next steps captured

At this point: Syncthing running on the desktop, `shared-games` folder created, GhostShip added. The Ally was next — a separate session to pair the devices and provision the other end.
