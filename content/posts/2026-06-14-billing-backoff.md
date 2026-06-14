---
title: "the billing backoff was set to 24h"
date: 2026-06-14
draft: false
categories: ["ai"]
tags: ["openclaw", "configuration", "agents", "gotcha"]
summary: "The default billingBackoffHours value means one brief Claude quota hit locks out all agents until the next day."
---

One of the bots went quiet. Then two. Then I realized they'd all gone dark around the same time.

The root cause was `auth.cooldowns.billingBackoffHours: 24` — the default. When Claude's subscription quota gets hit (even briefly, even late in the day), OpenClaw backs off from that auth provider for 24 hours. There's no automatic retry, no grace period. If all agents share the same fallback chain and none have a working provider left, they don't respond. They don't error. They just stop.

Changed it to 1. A brief cap hit now means a one-hour cooldown, not a lost day.

The fix I added alongside: an all-Claude fallback chain (sonnet → sonnet → opus) without any auto-fallback to a metered provider. Codex had been quietly entering the chain when Claude was capped — not as a deliberate backup, just next in the list. I hadn't noticed because the output looked the same. Switched to Claude providers only, shorter backoff, and a watchdog cron that alerts on sustained failures.

I don't know what the right value for `billingBackoffHours` is in general. One hour might be too short if you're against a hard cap and retrying just racks up failed turns. But 24 is definitely not right for a personal assistant stack where you're not watching the logs.
