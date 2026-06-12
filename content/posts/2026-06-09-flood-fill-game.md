---
title: "the flood fill is the game"
date: 2026-06-09
draft: false
categories: ["notes"]
tags: ["godot", "gamedev", "side-project", "steam"]
summary: "Built a warehouse loading dock puzzle in Godot 4. The whole mechanic is a BFS from the door that determines what freight can be reached."
---

The concept started as "3D Tetris for a garage" and didn't stay there long.

The problem with Tetris-as-metaphor is that rows disappear. Nothing disappears in a warehouse. You pack freight in and then — critically — you have to get it back out. Pack too dense and the outbound picker can't reach anything. That's the constraint.

So the mechanic: a flood fill from the loading door at every placement. Green cells are reachable aisle. Amber is enclosed dead space that nothing can reach but also isn't blocking anything. Red is stranded freight — items that no path from the door can get to. Place something that cuts off a path and anything behind it turns red immediately.

It's not about clearing. It's about access cost.

Built in Godot 4.6.3. The main loop is GDScript: grid, BFS per placement, mouse place/rotate/undo/clear, iso/top/head-on orthographic camera snaps. Toon shader with inverted-hull outlines because the grid geometry needed some visual weight to read clearly.

Exported as a native Linux binary, not flatpak. Flatpak adds sandbox friction to headless CLI export that isn't worth it when you're using Godot as a build tool — download the official binary, put it in `~/.local/bin`, done.

To get it into Steam as a non-Steam shortcut, I needed to write to `shortcuts.vdf` — a binary format in `~/.local/share/Steam/userdata/<id>/config/`. The `vdf` Python library handles it cleanly: read, edit, write. One gotcha: Steam rewrites `shortcuts.vdf` on shutdown. Write it while Steam is running and Steam wins. Shut Steam down first, write the file, leave Steam down.

v0 is a feel test. The question is whether "keep the aisles clear while freight arrives" is actually fun to play.
