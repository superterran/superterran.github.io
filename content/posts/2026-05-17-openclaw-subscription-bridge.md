---
title: "wiring openclaw to claude cli with subscription billing"
date: 2026-05-17
draft: false
categories: ["homelab"]
tags: ["openclaw", "claude", "mcp", "docker", "billing", "litellm"]
summary: "OpenClaw's native anthropic plugin bills against a metered pool; the claude-cli provider uses your Claude subscription instead. Three non-obvious pieces: an API key stripping wrapper, a silent per-agent model constraint, and an insecure WebSocket flag for same-host Docker networking."
---

OpenClaw supports two ways to connect to Claude: the native `anthropic` plugin (direct API) and the `claude-cli` provider (shells out to the `claude` binary). The billing difference matters: the `anthropic` plugin bills against a metered API usage pool; `claude-cli` uses the Claude subscription on the account the binary is logged in to. For a homelab running continuous agent sessions, the subscription path is substantially cheaper.

## The ANTHROPIC_API_KEY stripping wrapper

OpenClaw injects `ANTHROPIC_API_KEY` into the environment for its native plugin. The `claude` CLI binary sees this environment variable and treats it as a literal API key — which it isn't; it's an OAuth token in OpenClaw's format. The CLI rejects it and authentication fails.

Fix: a wrapper script at `/home/node/bin/claude` that unsets `ANTHROPIC_API_KEY` before exec-ing the real binary:

```bash
#!/bin/bash
unset ANTHROPIC_API_KEY
exec /usr/local/bin/claude-real "$@"
```

With the wrapper in `PATH` before the real binary, OpenClaw's `claude-cli` provider calls the wrapper, which clears the injected key, and the binary authenticates normally via its stored credential.

## Per-agent models.json is a silent constraint

OpenClaw allows per-agent model configuration via a `models.json` file. If `claude-cli` isn't listed as an available provider in that file, OpenClaw falls back silently to whatever provider is listed — it doesn't error, it just picks the next option. If the fallback is an unconstrained LiteLLM-routed model, requests that should be going to Claude end up elsewhere without any log warning.

The symptom: responses that seem weirdly inconsistent with Claude's behavior. Check `models.json` for the agent and confirm `claude-cli` is explicitly listed before assuming the billing or auth setup is broken.

## openclaw-mcp SSE bridge

The `openclaw-mcp` bridge exposes OpenClaw's internal channels API as MCP tools, accessible over HTTP/SSE. This lets Claude sessions (or other MCP clients) read and write to OpenClaw channels without going through the Slack surface. Useful for agent-to-agent coordination that doesn't need to be visible in Slack.

Configuration is minimal — point the MCP client at the bridge endpoint and provide a bearer token. The bridge itself runs as a Docker container alongside the OpenClaw gateway.

## alpine/openclaw is Debian

The `alpine/openclaw` Docker image name is misleading. Despite the namespace, the image runs Debian bookworm, not Alpine Linux. This matters when you're installing packages or debugging the container — `apk` doesn't exist, `apt` does. The namespace reflects the publishing org, not the base OS.

## OPENCLAW_ALLOW_INSECURE_PRIVATE_WS

When the `openclaw-mcp` bridge runs in a Docker container on the same host as the OpenClaw gateway, it connects to the gateway over a Docker network interface. Docker bridge networks use RFC 1918 addresses, but they're not loopback — the gateway's TLS configuration may reject non-loopback WebSocket connections by default.

Setting `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` in the bridge container's environment allows the WebSocket connection to proceed over the Docker network without TLS. Only appropriate for same-host Docker deployments where the traffic never leaves the machine.
