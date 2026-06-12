---
title: "coordinating multiple slack bots in shared channels"
date: 2026-05-24
draft: false
categories: ["homelab"]
tags: ["openclaw", "slack", "multi-agent", "bots", "coordination"]
summary: "Two AI agents writing duplicate artifacts in the same Slack channel. Prompt-level 'stay in your lane' doesn't work — it can't win a race. Structural fixes do."
---

OpenClaw runs multiple AI agents across a few Slack workspaces. Some channels have more than one agent with write access. The problem that emerged: in high-activity threads, two agents would respond to the same message in parallel and produce conflicting or duplicate artifacts — both writing the same document update, both posting the same answer. Prompt coordination ("don't respond if the other agent already has") is too slow. It can't win a race between two parallel agent turns.

The structural fixes that work:

## requireMention gate

Per-channel `requireMention: true` makes an agent fire in that channel only when directly `@`-mentioned. In any shared channel, designate one agent as the operational primary (responds freely) and make the other a consultant (responds only when tagged). A primary that never tags the consultant = no interference.

This is the core fix. Everything else is layered on top of it.

## Orchestrator / consultant pattern

The primary is also the orchestrator. When the consultant is needed, the primary decides when to tag them and with what specific ask. Single-head decision — no race condition because only the primary can initiate the consultant.

What makes this work in practice: the primary reads the thread first, determines what's needed, then either handles it or tags the consultant with a specific question. "What's your take on X?" not just a bare ping. The consultant does their job and exits.

## `slack_read_thread` as the mandatory first tool call

In any shared-bot channel, session memory is unreliable as a source of truth — each agent's session is isolated and doesn't include other agents' recent messages. The live thread is the only source of truth. Making `slack_read_thread` the mandatory first tool call before any action ensures the agent is working from current state, not a stale cached view.

Without this: an agent wakes up in a thread, misses what the other agent just did, and posts a duplicate.

## `[state]` pins

After any canonical state change (new artifact, owner change, action closed), post one line in thread:

```
[state] <summary>. Owner: <who>. Open: <what>.
```

Most recent wins. This is cheap — one line of text — and visible to both humans and agents on thread read. When an agent wakes up in an existing thread and reads back, the most recent `[state]` line gives it an anchor. No need to parse the whole thread's history to figure out where things stand.

## Cron session-key vs. isolated sessions

Some agents do work via scheduled cron tasks that touch active threads. The wrong pattern: `--session isolated`. An isolated cron task's work doesn't appear in the agent's thread session memory. The agent subsequently re-enters the thread with no memory of what the cron task just did.

The right pattern: `--session-key agent:<name>:channel:<id>:thread:<ts>`. Session-keyed cron tasks join the same session as the thread, so their work is visible in subsequent turns.

---

## The OpenClaw upgrade (2026.5.7 → 2026.5.19)

Ran the queued upgrade in the same session. One thing worth noting: the TrueNAS catalog moves past staged versions quickly. The runbook said target version `1.0.32` (2026.5.18); by the time the upgrade ran, the catalog only offered `1.0.33` (2026.5.19). The delta was small — a stable rollup plus one point release. Confirm the available version via `midclt call app.upgrade_summary <app>` immediately before running, not by trusting a hardcoded version in a runbook written a week earlier.

Post-upgrade all Slack sockets reconnected cleanly. The upgrade included a restart-drain fix that had been causing cron jobs to get interrupted by gateway restarts — confirmed by checking cron job error counts, which showed two consecutive failures on the dreaming pipeline. Manually triggered the failed job, confirmed it ran successfully, error count reset.

## Per-agent flutter tuning

When an agent in a Slack channel is "thinking," a typing indicator shows. With multiple agents in a channel, you want this to mean something — flutter only when there's actual content incoming, not on every wakeup.

In 2026.5.19, per-agent `typingMode` isn't configurable directly. The workable configuration: `reasoningDefault: "stream"` and `thinkingDefault: "medium"` per-agent, with `typingMode: "thinking"` in the global defaults. This makes the flutter appear while reasoning is streaming and stay quiet otherwise. Hot-reloads without a gateway restart.
