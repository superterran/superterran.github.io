---
title: "the guard was lying"
date: 2026-09-09
draft: false
categories: ["infra"]
tags: ["monitoring", "auth", "bash", "codex"]
summary: "An hourly auth-check script had been reporting OK for days while the token it guarded counted down toward actual expiry."
---

A script that guards a Codex auth token had been logging `OK — Codex token refreshed` on every run for days. Except the JWT expiry kept counting down the whole time: 203 hours left, then 131, then 107. Every check still said OK.

The script assumed that any `codex exec` call re-mints the token, the way most CLIs quietly refresh on use. Easy enough to test directly: with 68 hours left, ran a real `codex exec`, got a clean exit and real output, checked the expiry again. Unchanged. Codex only refreshes lazily, at or after the token actually expires, never before.

So the "OK" test — `hours_remaining > 48` — wasn't checking whether a refresh had happened. It was checking whether the old token still had life in it. That can only fail in the token's last two days, and by the time anyone noticed, the token backing every request had less than three days left.

Rebuilt the check around what actually matters. Read the expiry straight out of the JWT, no API call, no quota spent. Only spend a real `codex exec` once inside a tight window before expiry, so the lazy refresh fires against a throwaway call instead of a live one. Success now means the expiry timestamp moved forward, not that the last command happened to exit zero.
