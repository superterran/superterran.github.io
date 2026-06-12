---
title: "systemd timers for personal ai tasks"
date: 2026-03-28
draft: false
categories: ["homelab"]
tags: ["bazzite", "systemd", "automation", "claude", "plaud", "scheduling"]
summary: "Wiring up systemd timers for the Plaud sorter and a daily planner. Two patterns that come up every time: headless Claude needs --dangerously-skip-permissions, and weekday/weekend schedules fit in a single timer file."
---

The homelab accumulated a few recurring AI tasks that needed to run on a schedule rather than on-demand: sorting new Plaud voice note transcripts into the right folders, and generating a personal daily plan from recent notes and calendar context. Both ended up as systemd user timers on bazzite.

Two things that come up every time you wire Claude CLI into a systemd unit:

## `--dangerously-skip-permissions` for headless operation

Claude CLI in interactive sessions prompts before writing files. There's no TTY in a systemd unit, so those prompts hang indefinitely. The flag `--dangerously-skip-permissions` bypasses the approval flow. It's the right call for trusted automation running against your own files — just be specific about what you're pointing it at.

## `OnCalendar` supports multiple entries

For the daily planner, the schedule needed to be different on weekdays vs. weekends — 7 AM weekdays, 9 AM weekends. One timer file can carry both:

```ini
[Timer]
OnCalendar=Mon..Fri *-*-* 07:00:00
OnCalendar=Sat..Sun *-*-* 09:00:00
```

No need for two separate timer files or a wrapper script checking the day.

## Plaud sorter

The Plaud sorter is a batch job, not file-event-driven — it runs once daily and processes whatever's in the queue. Shell wrapper that invokes `claude` with the sorter skill, systemd timer set to 8 AM daily. Simple.

The plaud-ai-watch watcher (which fires on file events for individual note post-processing) was also migrated from OpenCode/Ollama to Claude CLI in the same session. The working directory pattern changed: `opencode run --dir <path>` doesn't have a direct equivalent, so the scripts were updated to use a subshell `cd` instead:

```bash
(cd "$VAULT_DIR" && claude -p "$prompt" --model "$MODEL" --dangerously-skip-permissions)
```

## Dashboard

All the services and timers across the fleet got collected into a single `Context/scheduled-tasks.md` in the vault — a manual but persistent overview of what's running where. When you've got services on bazzite and closet and launchd agents on the MacBook, a flat markdown table beats hunting through `systemctl list-timers` on three machines.

## One open pattern for macOS

Launchd plist templates for both tasks were added to the skill SKILL.md files. macOS launchd uses `StartCalendarInterval` for scheduled runs — different key names and format than systemd `OnCalendar`, but structurally the same idea.
