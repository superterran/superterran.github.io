---
title: "the madhabits switch-fallthrough"
date: 2026-06-30
draft: false
categories: ["notes"]
tags: ["museum", "python", "php", "recovery"]
summary: "Rebuilding madhabits.com from 2012 PHP source — and preserving a switch-fallthrough quirk that had always been part of the look."
---

Found the madhabits.com source in a 2012 NAS backup. The site used PHP with functions named `line()`, `makebg()`, `render()` to produce a hex-dump aesthetic — rows of monospace digits, a working theme switcher, the whole bit. The original MySQL database was in there too.

Rebuilt it as a Python generator for [the museum](https://museum.superterran.net), which keeps static snapshots of old projects. The port needed to match the original visual output exactly, so it stepped through the rendering logic character by character.

The switch statement in the original PHP handled the hex display. It had cases for `0` through `9` and then `F` — no `A` through `E`. So bytes with values 10–14 printed as `10`, `11`, `12`, `13`, `14` instead of `A`, `B`, `C`, `D`, `E`. Not a typical hex dump, but apparently that's what it always produced. Whether this was a mistake is lost to 2006.

The Python port reproduces it exactly. Changing the bug would make the reproduction less accurate.

The theme switcher still works. Pure client-side CSS class swap.

Live at [museum.superterran.net](https://museum.superterran.net).
