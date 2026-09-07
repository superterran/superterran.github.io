---
title: "retroarch keys saves to the rom file, not the game"
date: 2026-09-07
draft: false
categories: ["tools"]
tags: ["retroarch", "emulation", "sync", "debugging"]
summary: "A 'lost' save turned out to be a healthy sync job pointed at the wrong file the whole time."
---

A widescreen shortcut for Super Metroid looked like it had eaten a save. Desktop had progress, the handheld didn't, and cloud sync seemed like the obvious suspect.

It wasn't. The `.srm` file was identical — same md5 — on the desktop, the handheld, and the WebDAV remote, synced minutes before I went looking. The sync was working perfectly. The file it was syncing was just empty: every byte 0x00 or 0xFF, ten distinct values total.

The `.lrtl` runtime logs settled it before I'd even opened the ROM. The widescreen shortcut had a twelve-second play time and one launch, ever. It had never actually been played.

RetroArch names save files after the ROM's basename, filed under the core. The widescreen hack is a different file than the vanilla ROM — different name, so a fresh save slot, even though it's a patch of the same game and the SRAM layout is identical underneath. Two ROMs, same game, two saves, and only one of them had hours in it.

Fix was a straight copy: took the real save from the vanilla slot, wrote it into the widescreen slot on both machines and the remote. Checked the ROM sizes matched first — a widescreen patch changes the header and code, not memory layout, so SRAM is portable across the pair.

Then it happened twice more the same day, with different games, because it isn't really a bug — it's how RetroArch always works. One Super Mario World shortcut had three separately-named ROM files, all byte-identical, each with its own diverging save. Scored them by completion bits set and kept the one with the most progress.

I check whether the file being synced is actually empty now, before assuming sync is broken.
