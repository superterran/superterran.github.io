---
title: "three shared-games bugs, three root causes"
date: 2026-04-04
draft: false
categories: ["homelab"]
tags: ["bazzite", "steam", "syncthing", "bash", "gaming"]
summary: "Steam shortcuts weren't launching, the setup script was generating bad output, and a required ROM wasn't syncing. Three separate bugs, all with clear root causes."
---

After setting up a Syncthing-based shared game library, shortcuts added to Steam weren't actually launching. Debugging it turned up two independent bugs in the setup script and a third sync-scope problem with a required ROM file.

## Bug 1: VDF requires quoted exe paths

Steam's non-Steam shortcut format is a binary VDF file. The `exe` field for a non-Steam game needs to be a quoted path — `"/home/me/games/launch.sh"` not `/home/me/games/launch.sh`.

The setup script was writing unquoted paths. Steam accepted the VDF without complaint, showed the shortcuts in the library, and then silently failed to launch any of them. No error, just nothing happening on click. Once the exe paths were wrapped in double-quotes, everything launched.

## Bug 2: ANSI escape codes in `$()` capture

The `add_steam_shortcut` function was calling info/success helper functions inside a `$()` subshell to capture output. Those helpers printed colored status messages to stdout. The ANSI escape codes ended up embedded in the captured value, corrupting it.

Fix: redirect all info/success output to stderr with `>&2`. The calling context gets clean stdout, the terminal still shows the status messages, nothing is corrupted.

This is a variant of the "don't mix status output with return values" rule. It's easy to miss in bash because the failure mode is silent until you inspect what the variable actually contains.

## Bug 3: ROM not in sync scope

Ship of Harkinian (an Ocarina of Time PC port) needs a `baserom.nes` file to generate its game data on first launch. That file was sitting in a device-local directory on one machine, not in the Syncthing shared-games folder. The other machine never got it.

Fix: moved the file into the shared-games folder, symlinked it back from the XDG directory the game expects. Syncthing picks it up as part of the normal sync. One machine generates the game data; other machines get the ROM and can generate their own.

The same pattern — move the asset into the shared folder, symlink outward — applies to any game that needs a seed file that isn't bundled.

## One other thing

After editing Steam shortcuts, you need to restart Steam on all affected devices for the changes to fully apply. The VDF is read at startup; changes mid-session don't propagate.
