---
title: '"thinkingDefault: medium" silently means off'
date: 2026-06-12
draft: false
categories: ["ai"]
tags: ["openclaw", "claude", "agents", "gotcha"]
summary: "Setting thinkingDefault to medium in an OpenClaw agent config does nothing. The valid values are low and high; medium gets silently dropped to off."
---

The config key in question:

```json
"thinkingDefault": "medium"
```

This looked reasonable. More reasoning than `low`, less than `high`. Moderate. The sane default.

Except "medium" is not a valid value on Claude 4.6/4.7. The actual enum is `[low, high]` — there is no middle. OpenClaw accepts the config without complaint, silently downgrades to `off`, and your agents proceed with no extended reasoning at all.

I had this set across four agents for at least a few weeks. The agents were running fine — tool calls, normal responses — but none of the reasoning that `thinkingDefault` is supposed to enable was happening. 15+ turns per day per agent, all quietly in `off` mode.

Caught it during a fleet audit. The fix was one line in each agent config:

```json
"thinkingDefault": "low"
```

No error, no warning, no log line when the invalid value got swallowed. If you're running OpenClaw agents and have `thinkingDefault` set to anything besides `off`, `low`, or `high`, verify the actual model config it sends to the provider. Don't assume the gateway validated your input.
