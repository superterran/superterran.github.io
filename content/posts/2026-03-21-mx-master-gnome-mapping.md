---
title: "remapping mx master 4 buttons for gnome with solaar"
date: 2026-03-21
draft: false
categories: ["homelab"]
tags: ["bazzite", "gnome", "logitech", "solaar", "wayland", "input"]
summary: "Haptic thumb-rest button → GNOME Activities. Side gesture button held + scroll wheel → Control+plus/minus zoom. Solaar rules, Flatpak quirks."
---

Two remaps on the MX Master 4 under GNOME on bazzite: the haptic thumb-rest button should open the Activities overview, and the side gesture button held while scrolling should zoom in/out (closest mouse-wheel equivalent of pinch gesture on Linux).

## Solaar is a Flatpak here

On Bazzite the Logitech Solaar pairing/config tool is installed as a Flatpak. CLI operations go through:

```bash
flatpak run io.github.pwr_solaar.solaar <command>
```

Not a problem, just worth knowing before you try to call `solaar` directly and get a command-not-found.

## Button discovery

`solaar show` lists available divertable keys. On this MX Master 4 profile, both `Haptic` (thumb-rest) and `Mouse Gesture Button` (side paddle) show up as reprogrammable. The HIRES WHEEL is the high-resolution scroll wheel used in the zoom rule.

## The rules

Two rules in the Solaar config:

**Haptic → Activities:**
```
Key: Haptic
  KeyPress: Super_L
```

`Super_L` is what GNOME watches for the Activities shortcut. Direct and simple.

**Side button held + scroll → zoom:**
```
Mouse Gesture Button held:
  hires-scroll-mode: on
  HIRES WHEEL up:   KeyPress Control_L+equal   (zoom in)
  HIRES WHEEL down: KeyPress Control_L+minus   (zoom out)
Mouse Gesture Button released:
  hires-scroll-mode: off
```

The hold/release pair toggles hires scroll mode while the button is held and restores normal scroll on release. Without this, normal scrolling stays in hires mode after the first gesture.

## What I replaced

The previous config had a `Feature: GESTURE 2` rule that triggered too broadly — it matched behavior across multiple buttons and wasn't reliable for targeting specific inputs. Explicit `Key` conditions per button are more predictable.

## One quirk

Setting `Mouse Gesture Button` diversion to `Mouse Gestures` via CLI shows as `Diverted` when queried back, but actual behavior may depend on firmware and Solaar runtime interpretation. Worth testing each rule after a Solaar reload rather than assuming the stored value reflects exactly what will happen.
