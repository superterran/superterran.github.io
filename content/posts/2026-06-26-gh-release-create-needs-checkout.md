---
title: "gh release create still needs git"
date: 2026-06-26
draft: false
categories: ["tools"]
tags: ["github", "github-actions", "gh-cli", "gotcha"]
summary: "Setting GH_REPO in a GitHub Actions job isn't enough for gh release create — it still shells out to git internally, so you get 'fatal: not a git repository' without a checkout step."
---

Running `gh release create` in a GitHub Actions job with no checkout step. Set `GH_REPO=org/repo` — which is the documented way to point `gh` at a target repo from outside one. Job still died: `fatal: not a git repository`.

The assumption was that `GH_REPO` would make `gh` fully repo-aware. It does for most things. `gh release create` is different: it shells out to `git` internally at some point, and `git` wants an actual directory with a `.git/` folder, not an env var.

Fix: `actions/checkout@v4` at the top of the job, before the release step. Doesn't need a deep clone — `fetch-depth: 1` is fine. Just needs the directory context so `git` doesn't bail.

The error message is a dead end. "Not a git repository" points at your checkout setup, not at `GH_REPO`.
