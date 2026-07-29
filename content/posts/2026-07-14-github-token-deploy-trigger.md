---
title: "merged to main, still read-only"
date: 2026-07-14
draft: false
categories: ["infra"]
tags: ["github-actions", "ci", "deploy", "gotcha"]
summary: "GITHUB_TOKEN can't trigger other workflows. If your daily backup job cuts a release with it, that release won't fire your deploy pipeline."
---

A change got merged. The user said it wasn't working. Turned out the change wasn't deployed — just merged.

The deploy workflow only runs on `release: published`. Seemed fine. But there's a second thing to know: when a workflow creates a release using `GITHUB_TOKEN`, GitHub won't let that release trigger other workflows. It's an anti-recursion guard — same reason `push` events from `GITHUB_TOKEN` don't re-trigger CI.

The daily backup job was cutting automatic `backup-*` releases via `GITHUB_TOKEN`. Those releases fired every night without deploying anything. Not a bug — they weren't supposed to deploy anything. But there was a mistaken assumption that "a release got created" was equivalent to "the deploy ran."

The fix: to actually deploy, cut a release manually with `gh release create vX.Y.Z` (authenticated as a user or PAT), or `workflow_dispatch` the deploy directly. The `release: published` trigger only fires when the release creator is a real user/PAT token.

One thing this makes obvious in hindsight: the deploy trigger and the backup job both produce GitHub releases, but they're not the same kind of release. The convention was `backup-YYYY-MM-DD` vs `vX.Y.Z` — different naming, different purpose, different trigger behavior. Worth making that explicit in the runbook so you don't stare at a green merge and wonder why nothing happened.
