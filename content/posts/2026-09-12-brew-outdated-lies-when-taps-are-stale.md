---
title: "brew outdated lies when the taps are stale"
date: 2026-09-12
draft: false
categories: ["tools"]
tags: ["homebrew", "claude-code", "package-management", "linux"]
summary: "Claude Code nagged to update for months because Homebrew's taps hadn't been fetched since April, so brew outdated reported the ancient cask as current."
---

Claude Code kept nagging to update, every launch. Ran `brew upgrade --cask claude-code` — nothing happened. Ran `brew outdated` — empty list. "Up to date."

It wasn't. The installed cask was pinned at 2.1.112. Current was 2.1.261. A hundred and forty-nine versions behind, and Homebrew was completely sure there was nothing to do.

The taps hadn't been fetched since April 28th. `brew outdated` diffs the installed version against whatever's sitting in the local tap cache — it doesn't reach out and check anything unless a `brew update` forces the fetch first. No fetch, no diff, no problem, as far as brew was concerned. Claude's own in-app updater couldn't help either; it has no path to replace a brew-managed install, so it just printed the same nag on every launch and gave up quietly each time.

Fix was to stop routing through brew at all. Installed the native installer to `~/.local/bin/claude`, then `brew uninstall --cask claude-code`. Claude was brew's only package on that machine, so brew is now just sitting there empty — no taps left to go stale.

The native binary self-updates on its own, so this particular version of the problem shouldn't recur. But the failure mode is worth keeping around: a package manager reporting "current" only ever means current-against-its-last-fetch, not current-against-upstream. If nothing forces the fetch, "up to date" can be silently, indefinitely wrong.
