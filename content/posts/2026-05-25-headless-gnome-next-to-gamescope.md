---
title: "headless gnome next to gamescope"
date: 2026-05-25
draft: false
categories: ["homelab"]
tags: ["bazzite", "gnome", "gamescope", "sunshine"]
summary: "Sunshine and Gamescope stopped fighting once GNOME got its own headless Wayland session instead of sharing the physical display."
---

Sunshine had been losing a boot-time fight with Gamescope.

That made sense once I stopped treating it like a Sunshine problem. Gamescope owned the physical display. Sunshine wanted a desktop session it could stream. Letting both care about the same output was the messy part.

The cleaner shape was a second compositor:

```text
gnome-shell --wayland --no-x11 --headless --virtual-monitor 2560x1440@60
```

That gives Sunshine a GNOME Wayland session to target without touching the local Gamescope session. The TV can stay owned by Gamescope. The stream can point at the virtual monitor. No fake HDMI dongle, no boot race, no "which thing grabbed the display first" behavior.

The idle cost was small enough to ignore on the workstation, roughly a few hundred MB. GNOME's screenshot and eval APIs still hit their normal privacy guards, but that is not the interesting path here. Sunshine does not need GNOME Shell to become a remote-control API. It needs a Wayland session with a monitor.

The useful bug was in my capture script. It grabbed top-level systemd unit files but missed drop-in directories, which meant the live Sunshine override was not actually represented in the repo. That is exactly the kind of thing that makes an immutable-ish desktop feel reproducible until the first rebuild proves otherwise.

This is not enabled at boot yet. It needs one clean Moonlight test from another machine before it gets promoted from "works manually" to "part of the machine."
