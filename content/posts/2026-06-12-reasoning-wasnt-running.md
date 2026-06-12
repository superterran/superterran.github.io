---
title: "the reasoning wasn't running"
date: 2026-06-12
draft: false
categories: ["ai"]
tags: ["openclaw", "claude", "agents", "gotcha"]
summary: "A fleet audit found four agents had been running without extended reasoning for weeks. The config looked fine. Behavior looked normal. Nothing paged."
---

The config key that started it:

```json
"thinkingDefault": "medium"
```

I did an end-to-end audit of my agent stack after a model upgrade. Found this across four agents. "medium" reads as the sensible middle ground. Not max reasoning, not off.

Claude 4.6/4.7 doesn't have a medium. The valid values are `off`, `low`, `high`. OpenClaw accepted the config, dropped it to `off`, and didn't log anything.

Four agents. Weeks. Fifteen or more turns a day. No extended reasoning during any of it. The responses looked normal. There was no way to notice from behavior alone — the model is still competent at `off`, it's just not running the way you configured it.

Found a second version of the same failure while I was in there: the deploy script was defaulting to a model alias that had gotten misconfigured during a prior upgrade. Agents coherently running on a different model than the one I thought I'd pinned.

Both caught by reading what the gateway was actually sending, not by noticing degraded output. Not by an alert.

AI config rot doesn't fail loudly. It just runs at a different capability level than the one you set, and says nothing about it.
