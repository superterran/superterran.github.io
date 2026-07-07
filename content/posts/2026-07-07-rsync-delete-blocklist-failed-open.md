---
title: "rsync --delete and the blocklist that failed open"
date: 2026-07-07
draft: false
categories: ["infra"]
tags: ["rsync", "gotcha", "deploy", "sync"]
summary: "A deploy step used rsync --delete with an exclude-from blocklist. The blocklist silently failed to apply. rsync deleted the destination."
---

A deploy script was keeping a directory in sync with rsync using `--delete`. The approach: maintain a blocklist of paths to preserve even when they're absent from the source — things managed outside the sync, like clones and config that only live on the destination.

The blocklist was passed via `--exclude-from`. It worked fine on happy paths. At some point the loading logic failed silently, the exclusion list came back empty, and rsync saw a clean source against a destination full of extra files. `--delete` did what it says.

The fix: invert the model. Instead of an exclusion list, maintain an explicit allowlist of what belongs in the destination. rsync only touches the canonical set; anything outside that set is either not synced there or ignored. If the allowlist fails to load, the sync doesn't run — it doesn't wipe.

Blocklists fail open. Allowlists fail closed.

This is obvious in security contexts and easy to overlook in ops tooling. An rsync blocklist feels safe because the list is right there and works on every normal run. It just doesn't hold under the failure case you weren't thinking about when you wrote the script.

The other thing worth noting: `--delete` is quiet when it does large-scale removal. It doesn't prompt, doesn't summarize what it removed. If you're relying on it with an exclusion list, you're relying on that list to be present and loaded correctly every single time.
