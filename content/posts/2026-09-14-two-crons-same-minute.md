---
title: "two crons at the same minute, only one runs"
date: 2026-09-14
draft: false
categories: ["infra"]
tags: ["cron", "scheduling", "openclaw", "automation"]
summary: "A cron job that looked broken was actually losing a race against another job scheduled to the same minute on the same agent."
---

`AbortError, durationMs=3`. That's what a scheduled job logged the moment its window opened, and it told me nothing about what was actually wrong.

I run a handful of automations through OpenClaw's cron system, each scoped to an agent. Two of mine happened to land on the same minute in UTC, on the same agent. I didn't do this deliberately. They'd been added weeks apart, and I was thinking in Eastern time when I set them, not UTC.

Both jobs fire, but there's one serialized backend per agent underneath. The first job grabs it. The second one queues. By the time the first job finishes and the slot frees, the second job's own execution window has already closed, so it gets aborted immediately, `durationMs=3`, the instant it's handed a slot it can no longer use.

The symptom pointed the wrong direction entirely. It looked like a flaky plugin, something timing-related and environmental, because the failure only ever showed up on days both jobs were scheduled to run. That felt random unless you had both schedules open side by side. Converting every job's local time to UTC and diffing them line by line was the only way to actually see it: same agent, same minute, every time.

Fix was boring once the cause was clear. Move one job ten minutes later. No retries, no backend change, nothing clever. Boring is the right outcome for a bug that spent a week looking exotic.
