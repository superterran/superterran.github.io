---
title: "steam shortcuts wanted quoted paths"
date: 2026-05-25
draft: false
categories: ["tools"]
tags: ["steam", "syncthing", "games", "shell"]
summary: "The shared-games shortcut generator was writing valid-looking Steam VDF, but the launcher paths still needed quotes."
---

The shared-games scripts were adding shortcuts to Steam. The shortcuts appeared. They just did not launch.

That is worse than a loud failure because it looks like the hard part worked. The VDF file existed, Steam read it, the entries showed up in the library, and clicking them did nothing useful.

The first bug was simple: Steam wanted the executable path wrapped in quotes inside the shortcut VDF.

The second bug was more annoying. The helper that added shortcuts printed status output to stdout while another function was capturing its return value with `$()`. That meant nice colored logging leaked into a value that was supposed to be data.

The fix was not clever:

```text
print human status to stderr
return machine values on stdout
quote executable paths in the VDF
```

The third issue was a sync boundary. One ROM-derived file was still sitting in a device-local config directory, so the desktop had it and the handheld did not. The fix there was to move the file into the shared-games tree and leave a symlink where the app expected it.

All three bugs had the same shape: the automation assumed the filesystem and process boundaries were cleaner than they were. Steam was parsing a format with its own quoting rules. Shell capture was mixing UI text with data. Syncthing was only syncing exactly the tree I told it to sync.

None of these are big bugs. They are the little ones that make a "shared library" feel haunted until you trace each boundary and stop assuming the next tool shares your idea of obvious.
