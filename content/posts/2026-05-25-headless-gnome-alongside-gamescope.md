---
title: "headless gnome alongside gamescope on bazzite"
date: 2026-05-25
draft: false
categories: ["homelab"]
tags: ["bazzite", "gnome", "gamescope", "sunshine", "wayland"]
summary: "Running a headless GNOME Wayland session concurrently with Gamescope so Sunshine can stream a desktop without display contention."
---

Sunshine needs a Wayland compositor to capture from. Gamescope already owns the physical display. They can't share it, and the workaround I'd been running — disabling Sunshine at boot — isn't a real solution.

The actual fix: a second Wayland server, running headless.

```bash
gnome-shell --wayland --no-x11 --headless \
  --virtual-monitor 2560x1440@60 \
  --wayland-display wayland-headless
```

This exposes a socket at `/run/user/1000/wayland-headless` that Sunshine can target. Gamescope keeps the physical display entirely to itself. No contention, no boot flicker.

Mutter 49.3 (Fedora 43 / bazzite) handles the headless virtual monitor without complaint, including on NVIDIA 595.58.03 — which I wasn't confident about going in. Idle memory lands around 230–300 MB.

The service lives in a systemd user unit, backed by a launcher script in the bazzite IaC repo so it survives reinstalls.

One thing that caught me: `just capture-services` only snapshots top-level `*.service` and `*.timer` files. It skips drop-in directories entirely. Sunshine has an override under `sunshine.service.d/` that sets the Wayland display target. That override wasn't getting captured, so the repo looked complete and wasn't.

The service is running manually for now, pending an end-to-end Sunshine → Moonlight test. Once that verifies, it goes in as a proper boot-time dependency.
