---
title: "real-time obsidian vault backup with inotifywait and git"
date: 2026-03-14
draft: false
categories: ["homelab"]
tags: ["obsidian", "systemd", "git", "inotifywait", "backup", "bazzite"]
summary: "A polling git backup every hour is fine until you need the last 59 minutes. inotifywait watches for file events and pushes within 30 seconds of any change."
---

A timer-based vault backup running every hour leaves up to 59 minutes of work unprotected. The better approach: watch for file events and push when something actually changes.

## inotifywait as the backbone

`inotifywait` from the `inotify-tools` package watches a directory for events — creates, modifies, moves. The backup script runs in a loop: wait for an event, debounce briefly, run `git add -A && git commit && git push`. Latency between a vault change and a backup commit is typically under 30 seconds.

```bash
inotifywait -r -m -e create -e modify -e moved_to --format '%w%f' "$VAULT_DIR" | while read file; do
  # debounce
  sleep 5
  # drain remaining events
  while read -t 1 file; do :; done
  # commit and push
  git -C "$VAULT_DIR" add -A
  git -C "$VAULT_DIR" commit -m "auto-backup $(date -u +%Y-%m-%dT%H:%M:%SZ)" --allow-empty-message
  git -C "$VAULT_DIR" push
done
```

The debounce matters: Obsidian writes multiple small changes during a save. Without it, you'd get a push for each file in a bulk operation. The `read -t 1` drain loop after the sleep catches any events that queued up during the debounce window.

## systemd unit: paths with spaces

The vault is at `~/Documents/Cloud Vault/` — spaces in the path. systemd `ExecStart` and `EnvironmentFile` handle spaces poorly. Two options:

1. Quote with `\x20` escaping in the unit file
2. Create a symlink from a no-space path (`~/Context`) to the vault directory, and use the symlink in the unit file

Option 2 is more readable. The symlink survives Obsidian Sync, stays portable across machines.

## HTTPS over SSH for the git remote

The vault repo uses HTTPS with `credential.helper=store` rather than SSH. The reason: SSH key auth requires `IdentityAgent` configuration (`%d/.1password/agent.sock` for 1Password SSH), and getting that right inside a non-interactive systemd service is more work than HTTPS + a stored credential. For a personal vault repo, stored HTTPS credentials are acceptable.

One thing: `gh`'s OAuth token doesn't have the `workflow` scope by default. If the vault has `.github/workflows/` files, pushing via the stored gh credential will fail. Use a PAT with `workflow` scope, or avoid pushing workflow files from the watcher.

## The old approach

The previous setup was a systemd timer firing every hour running a similar git push. Replacing the timer with an always-on watcher service means: no polling interval to tune, no "did the change land before or after the last run" ambiguity, no accumulated work lost when the timer fires just after a session ends.

The watcher script lives in the vault itself (`.scripts/watch-and-push.sh`), so it's backed up alongside the content it protects.
