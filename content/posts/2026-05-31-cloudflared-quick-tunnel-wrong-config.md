---
title: "cloudflared quick tunnel, wrong config"
date: 2026-05-31
draft: false
categories: ["infra"]
tags: ["cloudflare", "tunnel", "networking", "gotcha"]
summary: "Quick tunnels pick up ~/.cloudflared/config.yml if it exists, and route through your named tunnel\'s ingress instead of the trycloudflare URL."
---

Ran `cloudflared tunnel --url localhost:1313` on a machine that also has a named Cloudflare tunnel configured at `~/.cloudflared/config.yml`. Got a `trycloudflare.com` URL back. Opened it. `404 Not Found`.

What\'s happening: cloudflared finds the config file and treats the process as a named tunnel invocation. The `trycloudflare.com` URL gets minted fine, but traffic routes through the named tunnel\'s ingress rules — which have no entry for `localhost:1313`. The URL is real, the routing is wrong.

Fix:

```
cloudflared --config /dev/null tunnel --url localhost:1313
```

No config file, no named tunnel context. The quick tunnel runs clean.
