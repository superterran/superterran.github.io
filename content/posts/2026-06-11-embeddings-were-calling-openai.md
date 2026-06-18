---
title: "the embeddings were calling openai"
date: 2026-06-11
draft: false
categories: ["ai"]
tags: ["openclaw", "ollama", "embeddings", "homelab"]
summary: "OpenClaw's memory search was silently hitting a metered OpenAI endpoint. Quota ran out, semantic memory degraded to keyword search, nothing complained."
---

Found this during a routine health sweep:

```
openai embeddings API: insufficient_quota — 429
```

I'd been running the stack on subscriptions — no API keys, no metered access, that was the whole point. The 429 didn't fit.

Turned out OpenClaw's memory search defaults to an OpenAI embeddings endpoint, not a local one. There's a `local` provider option but it needs a native module that isn't bundled. Everything else in the stack was subscription-routed; embeddings had been quietly calling OpenAI the whole time.

When the quota ran out, it didn't alert. It degraded to keyword search and kept running. Responses still came back. Memory still sort-of-worked. I only caught it because the 429 appeared in logs during a sweep I was already doing for other reasons.

Fix: point it at an Ollama instance instead.

```json
"memorySearch": {
  "provider": "ollama",
  "model": "nomic-embed-text",
  "baseUrl": "http://your-ollama-host:11434"
}
```

Pull the model if you don't have it:

```sh
ollama pull nomic-embed-text
```

Restart, re-index. OpenClaw sweeps all agents automatically; a few seconds per agent for a few hundred chunks. No more 429s, semantic memory restored, nothing leaves the LAN.

I pinned the Ollama config into my deploy script so it doesn't regress on the next redeploy.

The part that sticks: the failure looked like normal operation. If I hadn't been in the logs for a different reason, this would have run degraded indefinitely. That's the failure mode I keep running into with self-hosted AI stacks — things that break silently into a slightly worse version of themselves.
