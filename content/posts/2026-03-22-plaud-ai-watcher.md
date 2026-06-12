---
title: "auto-processing voice notes with inotifywait and claude"
date: 2026-03-22
draft: false
categories: ["homelab"]
tags: ["bazzite", "plaud", "obsidian", "automation", "systemd", "ai"]
summary: "A Plaud recorder drops markdown transcripts into the Obsidian vault. inotifywait watches for them, debounces, and hands each one to Claude to add a summary and action items."
---

The Plaud recorder syncs transcripts into the Obsidian vault as markdown files. They arrive verbatim — raw transcription, no structure. The goal was to automate post-processing: add a summary and action items without touching the transcript itself.

## The pipeline

`inotifywait` watches the vault's Plaud folder for create and modify events. When a file lands or changes, the watcher debounces (to avoid processing mid-sync), hashes the current file state to detect whether anything actually changed, and calls the AI if it did.

The AI step is just `opencode run --file <note> -- <prompt>`. The prompt is in a rules file in the skill directory, versioned with everything else. Outputs go to a log directory for inspection.

The avoidance logic matters: without hashing the state before and after, a modified event from the AI's own writes would trigger another run, which would trigger another event, looping indefinitely.

## What the AI does to each note

The rules baseline:
- Preserve all frontmatter and the `file_id` field
- Don't touch the transcript content
- Add or maintain a `## Summary` section
- Add or maintain a `## Action Items` section
- All edits are idempotent — rerunning on the same note produces the same result

Idempotence is important here. The watcher may see multiple events for the same note across sync cycles. The AI shouldn't pile up duplicate summaries or action items on re-runs.

## systemd and the spaces-in-path problem

The vault path is `~/Documents/Cloud Vault/...` — spaces in a directory name that show up in `EnvironmentFile` and `ExecStart`. systemd handles these poorly in unit files.

Fix: a `/home/me/Context` symlink to the vault's Context directory, used in all unit file paths. Stable, no spaces, everything resolves correctly.

## One opencode CLI gotcha

`opencode run --file <path> -- <prompt>` requires the `--` separator between `--file` and the prompt text. Without it, `opencode` tries to interpret the prompt words as additional file paths and fails.

## Plaud Sync plugin

The Plaud Sync Obsidian plugin handles the transcript delivery side — pulls notes from the Plaud API and drops them into the vault folder. Building from source (it wasn't in the community plugin registry) and dropping the build artifacts into the vault plugins directory.

The plugin stores its API token through Obsidian's secret storage API. There's no CLI path to inject it — it has to be entered through the plugin settings UI in the Obsidian app.

## End state

The watcher runs as a systemd user service, starts on login, restarts on failure. Drop a transcript into the folder and within a few seconds it has a summary and action items. The whole skill — watcher script, one-shot manual trigger, rules file, config — lives in `Context/skills/plaud-ai-watch/` in the vault, versioned alongside everything else.
