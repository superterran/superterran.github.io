---
title: "requireMention and threads"
date: 2026-05-30
draft: false
categories: ["mcp"]
tags: ["openclaw", "slack", "agents", "configuration"]
summary: "Per-channel requireMention gates top-level messages but not thread continuation. The companion setting is easy to miss."
---

Set `channels.slack.<channel>.requireMention: true` on an agent and it fires only when tagged. That's correct for new top-level messages. For threads, there's a separate rule.

Once a bot has replied in a thread, OpenClaw treats it as an active participant. Subsequent messages in that thread can re-trigger it whether or not they mention it — the per-channel mention gate only applies to the initial entry into a thread. It doesn't apply to continuation.

The companion: `channels.slack.thread.requireExplicitMention: true` at the top level of config (not per-channel). That setting extends the mention requirement into thread continuation. Without it, `requireMention` is half-functional: effective for keeping a bot quiet in top-level messages, invisible to everything happening inside an active thread where it's already participated.

Found this while debugging why one bot kept firing on messages addressed to a different bot. The `requireMention` gate had been set, the channel config looked right, and the bot was still responding. Took a while to notice the pattern: every firing message was inside a thread where the bot had replied once earlier.

Setting `thread.requireExplicitMention: true` at the top level applies to all agents and all channels simultaneously. Any channel that already has `requireMention` gets the thread gate as a companion automatically.
