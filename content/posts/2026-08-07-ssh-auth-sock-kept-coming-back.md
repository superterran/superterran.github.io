---
title: "SSH_AUTH_SOCK kept coming back"
date: 2026-08-07
draft: false
categories: ["homelab"]
tags: ["ssh", "bazzite", "1password", "dotfiles"]
summary: "The ssh agent helper unset SSH_AUTH_SOCK when it found a dead socket. .bashrc re-exported it one line later. Warning on every shell open."
---

The 1Password SSH agent socket only exists when the desktop app is running. bazzite runs as a Game Mode box most of the time. Desktop app doesn't run. So `SSH_AUTH_SOCK` pointed at a dead socket — warning on every shell:

```
Warning: 1Password SSH agent socket exists but not responding
```

The helper script handled this correctly: test the socket with `ssh-add -l`; if dead, `unset SSH_AUTH_SOCK` and proceed without an agent.

But the warning kept happening.

The problem: `.bashrc` had a block near the bottom — written there by the bazzite setup repo — that unconditionally exported the socket path:

```bash
export SSH_AUTH_SOCK=~/.1password/agent.sock
```

The helper ran, detected the dead socket, unset the variable. `.bashrc` continued to the next block and set it back. The test result was immediately discarded.

Fix was three places: the helper now tests *before* setting anything instead of after, `.bashrc` no longer overrides it, and the source file in the bazzite setup repo was updated so running setup again won't re-introduce the unconditional export.

The actual git SSH auth was routing through `IdentityFile` with `IdentityAgent none` in `~/.ssh/config` — the agent socket wasn't load-bearing for commits. The warning was cosmetic.

Still. A helper that detects a failure state and cleanly handles it is only useful if nothing downstream immediately reverses the handling.
