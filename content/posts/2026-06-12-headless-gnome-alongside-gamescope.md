---
title: "headless gnome alongside gamescope"
date: 2026-06-12
draft: false
categories: ["homelab"]
tags: ["bazzite", "gnome", "gamescope", "sunshine", "wayland", "systemd"]
summary: "Gamescope owns the display at boot, Sunshine wants a Wayland session. The fix: give Sunshine its own headless GNOME compositor with a virtual monitor that doesn't touch any physical output."
---

Gamescope owns HDMI-A-2 from the moment the desktop session starts. Sunshine, which I'd been wanting to use for game streaming via Moonlight, needs a Wayland compositor to attach to. The two were mutually exclusive at boot. My previous "fix" was just disabling Sunshine's autostart.

The actual fix is to give Sunshine its own session — a headless GNOME compositor running a virtual 2560x1440 monitor, no physical display, no interference.

## How it works

`gnome-shell` in mutter 49.3+ supports running fully headless with a virtual monitor:

```bash
gnome-shell \
  --wayland \
  --no-x11 \
  --headless \
  --virtual-monitor 2560x1440@60 \
  --wayland-display wayland-headless
```

This launches a complete GNOME Wayland compositor that exposes a socket at `/run/user/1000/wayland-headless`. Any app that needs a display can connect to that socket. Sunshine, once wired up, would target `WAYLAND_DISPLAY=wayland-headless` instead of the Gamescope display.

Runs at about 230–300 MB idle. No display contention with Gamescope because it owns no physical output.

## systemd unit

The service lives in `~/repos/bazzite/config/systemd/user/gnome-headless.service`:

```ini
[Unit]
Description=Headless GNOME Wayland session (for Sunshine streaming)

[Service]
Type=simple
Restart=on-failure
ExecStart=%h/repos/bazzite/bin/gnome-headless-session.sh

[Install]
WantedBy=default.target
```

A few intentional choices here:

**`WantedBy=default.target` not `graphical-session.target`** — The user's graphical session belongs to Gamescope. I explicitly don't want this enrolled in that session graph. `default.target` with `loginctl enable-linger` is the right anchor for a parallel session.

**`Type=simple`** — gnome-shell doesn't call `sd_notify`, so `notify` type would stall indefinitely. `simple` is correct, and `Restart=on-failure` handles crashes without restarting on clean exit.

**Not enabled at boot** — Sunshine end-to-end verification first. The memory cost is real, and I have a history of Sunshine-vs-Gamescope boot flicker.

## Verifying it works

After `systemctl --user start gnome-headless`, check the journal:

```
Added virtual monitor Meta-0
```

Then:

```bash
gdbus call --session \
  --dest org.gnome.Mutter.DisplayConfig \
  --object-path /org/gnome/Mutter/DisplayConfig \
  --method org.gnome.Mutter.DisplayConfig.GetCurrentState
```

Reports `MetaVirtualMonitor 2560x1440@60`, scaling 1.0–2.66x available. Launching `nautilus` with `WAYLAND_DISPLAY=wayland-headless` confirmed hardware-accelerated EGL-Wayland rendering via NVIDIA 595 — `lsof` showed it linked against `libnvidia-egl-wayland.so`.

## Things that surprised me

**No `dbus-run-session` wrapper needed.** A headless gnome-shell started under systemd user units inherits the user session D-Bus correctly. I expected to need a wrapper.

**`org.gnome.Shell.Screenshot` returning `AccessDenied` is not a failure.** It's a privacy guard. Sunshine uses Wayland protocols directly, not the Screenshot API.

**`just capture-services` doesn't capture drop-in directories.** My IaC recipe handles `*.service` and `*.timer` at the top level but skips `systemd/user/<service>.d/` directories. The live `sunshine.service.d/display.conf` wasn't in the repo — a silent gap.

**User extensions apply across both sessions.** The headless gnome-shell auto-applied a pending update for `quake-terminal` on first launch. Makes sense in retrospect; user-level extensions aren't scoped to a specific compositor.

## How it ended

After all of this, Doug (me) decided not to wire Sunshine to the headless session. The simpler model — just mirror whatever's on HDMI-A-2 — works fine, and switching sessions is one script away. The headless gnome-shell work is in the repo and confirmed working; it's just not active. Future use case is Playwright-over-Moonlight: run a browser session in the headless compositor, stream it, watch Claude drive it remotely. That experiment is parked.

The infrastructure is there. Sometimes that's enough.
