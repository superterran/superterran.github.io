---
title: "bootstrapMaxChars was silently truncating agent instructions"
date: 2026-06-16
draft: false
categories: ["ai"]
tags: ["openclaw", "agents", "configuration", "gotcha"]
summary: "OpenClaw's bootstrapMaxChars defaults to 12k; any AGENTS.md over that limit gets cut off on every turn with no warning."
---

Noticed one of the agents behaving weirdly — not quite following the lane rules set out in its AGENTS.md. Dug in and found the file was 14k. The gateway was cutting it to 12k on every turn without a warning anywhere.

`bootstrapMaxChars` defaults to `12000`. When your AGENTS.md exceeds that limit, the gateway injects whatever fits, drops the rest, and moves on. No error in the session logs, no truncation marker in the injected prompt. The agent just runs with a partial set of instructions it has no way to know is partial.

Checked the other agents. Three more were over the limit: 16k, 15k, 17.5k. All silently truncating, every turn, probably for weeks.

The fix is a one-liner:

```json
"agents": {
  "defaults": {
    "bootstrapMaxChars": 20000,
    "totalMaxChars": 40000
  }
}
```

You can also just trim the AGENTS.md. The one that hit 14k had lane-detail prose that probably belonged in the per-channel `systemPrompt` config anyway.

What made this annoying to diagnose: "agent isn't following instructions" is an ambiguous failure mode. Could be the model, could be the prompt, could be that the instructions were never fully delivered in the first place.
