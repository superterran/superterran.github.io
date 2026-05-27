---
title: "sunshine resolution list and what it doesn't capture"
date: 2026-05-27
draft: false
categories: ["homelab"]
tags: ["sunshine", "gamescope", "bazzite", "streaming", "moonlight"]
summary: "Adding custom resolutions to Sunshine's config exposes them in Moonlight — but with capture=kms, the stream is the HDMI plane, not the resolution you picked."
---

Added iPad Pro and ultrawide resolutions to Sunshine's `sunshine.conf` on Bazzite. Moonlight immediately showed them as selectable. The stream continued to look exactly like before: pillarboxed or letterboxed depending on what the TV reports, Gamescope FSR-scaling to fill the output.

The `resolutions` and `fps` arrays in `sunshine.conf` control the advertised list. That's what Moonlight displays when you pick a resolution. It doesn't change what Sunshine captures.

With `capture=kms`, Sunshine is reading the KMS plane — the actual signal going to the physical display. Gamescope FSR-scales everything to fill that output before it ever hits the plane. So what the stream sees is always the HDMI dimensions, bounded by what the TV declares in its EDID. Ultrawide and iPad-Pro aspect ratios aren't in most TV EDIDs; they'll silently fall back to the nearest supported mode.

I went looking for a runtime way around this. `gamescopectl` exposes an IPC socket and can change the refresh rate live. Resolution isn't on the menu — that's set at session init, not via IPC. The Steam Deck's per-game internal resolution swap goes through XWayland atoms that Steam controls; it's not an external tool you can call.

The workaround I built: a script that writes new `SCREEN_WIDTH/HEIGHT` into the environment config, then calls `loginctl terminate-session`. Autologin + linger re-fires gamescope at the new geometry. Sunshine's KMS capture reacquires after a few-second black frame. This works for modes the EDID supports — 1080p, 1440p, 4K — and fails quietly for anything outside it. It's exposed as a Moonlight "app" rather than a setting, which is awkward, but it's the only knob that actually changes what gets captured.

The actual architectural answer is `--virtual-connector-strategy` on gamescope. That flag creates a virtual KMS connector at an arbitrary size that Sunshine captures independently of the HDMI output. Aspect-correct streaming to any client resolution, no session restart required. Haven't built that yet.
