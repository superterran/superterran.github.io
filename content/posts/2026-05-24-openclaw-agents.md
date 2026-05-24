---
title: "five agents on a closet nas"
date: 2026-05-24
draft: false
categories: ["mcp"]
tags: ["openClaw", "agents", "truenas", "slack"]
summary: "Running five OpenClaw agents on a home NAS. The isolation pattern is per-MCP-surface, not per-tool-toggle. This post is written by one of them."
---

[OpenClaw](https://openclaw.ai) is an agent gateway. Give it personas, Slack credentials, and MCP server configs, and it runs Claude Code instances that behave like named people in your Slack workspace. Each persona can call tools, post messages, handle DMs, and run scheduled tasks.

I have five running on the TrueNAS SCALE box in my closet:

- **Hobbs** — personal ops. Calendar, homelab, household coordination, and the cron that reads session logs and opens PRs against this repo.
- **Sloane** — career/writing. Public-voice content, positioning, the other blog.
- **Bea** — shared household surface. Coordinates with my fiancée's Slack presence.
- **Familiar** — D&D companion. Reads campaign data via the D&D Beyond MCP.
- **Ember** — family-side, on a separate Slack workspace.

The isolation model is per-MCP-surface, not per-tool-toggle. Hobbs gets the full personal-ops surface: vault access, knowledge graph, wellbeing (read-only), task dispatch. Familiar gets only the D&D MCP and a separate knowledge graph project. Bea gets her own knowledge project but not the sensitive personal surfaces. You pick what each agent can see by configuring which MCPs they load, not by flipping individual tool permissions at runtime.

The compute is at Anthropic — the gateway just handles orchestration and message routing. A 4-core Celeron can run this fine.

Coordination between agents in shared channels was the part that needed explicit design. Hobbs and Bea both participate in household Slack channels. Without a mention gate, they'd both fire on any message and produce duplicate replies. The solution was `requireMention: true` in the channel config for the shared channels: Bea is the default for those; Hobbs only fires when @-mentioned. In the channel where Hobbs runs alone, mention gates are off.

This post was drafted by Hobbs, opened as a PR, and merged by me. The cycle is the point — getting to a state where the editorial pass is clean enough to trust without reading every draft first.
