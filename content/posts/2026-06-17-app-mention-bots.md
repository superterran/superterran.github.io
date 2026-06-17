---
title: "app_mention doesn't fire when a bot sends it"
date: 2026-06-17
draft: false
categories: ["ai"]
tags: ["openclaw", "slack", "agents", "gotcha"]
summary: "app_mention only fires for human senders. Bot-to-bot @-mentions route as message.groups instead, and require a different scope."
---

Set up a handoff where one agent @-mentions another in a private Slack channel. Expected the second agent to receive an `app_mention` event and respond. Nothing came through.

Checked the event payloads. The sending agent's message was arriving at Slack correctly — visible in the channel, no errors. The receiving agent's event log was empty.

The issue: `app_mention` only fires when the @-mention comes from a human user. When a bot sends the message, Slack routes it as `message.subtype=bot_message` under the `message.groups` event type. The `app_mention` subscription never sees it.

The fix requires two things: subscribe to `message.groups` in the Slack app's event subscriptions, and add `groups:history` to the bot token scopes. Without the scope, the event subscription can't deliver messages from private channels even if it's wired up.

Neither is obvious from the Slack API docs, which describe `app_mention` as the canonical way to receive mentions — but don't clearly flag the assumption that the sender is a human. If you're building multi-agent setups where bots hand off to each other by mention, this will burn you.
