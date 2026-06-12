---
title: "routing claude through litellm on truenas (and debugging a flaky bridge)"
date: 2026-05-20
draft: false
categories: ["homelab"]
tags: ["closet", "truenas", "litellm", "claude", "docker", "openai-api"]
summary: "LiteLLM + Postgres on closet to centralize AI routing across devices. The claude subscription bridge worked 40% of the time — then didn't, then did. Two red herrings and a version mismatch."
---

The goal was to move the AI routing stack from bazzite-desktop to closet so any device on the LAN could hit the same OpenAI-compatible endpoint, billed against Claude and ChatGPT subscriptions rather than per-token API pricing. LiteLLM as the router, a thin claude-bridge container for subscription auth, Postgres so the model list endpoint works.

Phase 1 (LiteLLM + Postgres) came up clean. Phase 2 (the bridge) took most of the session to debug.

## LiteLLM + Postgres

LiteLLM without a database returns a 400 on `/v1/models` — "No connected db" — which breaks Open WebUI's model fetch entirely. Postgres is mandatory if you want a model list. The compose stack is two services: LiteLLM and a Postgres sidecar sharing a ZFS-backed data volume.

First boot ran 122 Prisma migrations from scratch, taking about five minutes. Subsequent boots reconnect to the existing schema. Not a problem in practice but worth knowing if you're wondering why it's slow the first time.

First-try OOM: `mem_limit: 768m` got the container SIGKILL'd during the migration run (exit 137). Bumped to 1536m. Cold-start migration needs roughly 1 GB; steady-state settles around 600–800 MB. The limit can come back down after a few days of real use to find the actual headroom.

After the startup, `/v1/models` returned the configured aliases, `/v1/chat/completions` proxied through end-to-end. Phase 1 done.

## The claude bridge: 40% reliability

The bridge uses `claude-code-openai-wrapper` — a thin Python server that runs the Claude subscription CLI as a subprocess and presents an OpenAI-compatible API. On bazzite, running natively, this works reliably. In a container on closet, it was failing about 60% of the time with:

```
"[Request interrupted by user]"
```

Complete response with `finish_reason: stop`, 7 completion tokens, Claude's own interrupt-signal string. Not an HTTP error — a successful API call returning garbage.

### Red herring 1: shared session state

First theory: the bridge was mounting the same `~/.claude/` directory used by OpenClaw. OpenClaw writes session state to `~/.claude/projects/` continuously. Maybe the Claude CLI inside the bridge was detecting an in-progress session and returning an interrupt to avoid collision.

Fix: isolated the bridge to its own private `~/.claude/` directory, seeded from the OpenClaw credentials at container startup. Rebuilt, re-tested.

Result: worse. Reliability dropped. Shared session state wasn't the problem, but the isolation is still good hygiene and stayed in place.

### Red herring 2: PID 1 signal handling

Second theory: Python as PID 1 in a container doesn't handle SIGCHLD correctly by default. The SDK spawns the Claude CLI as a subprocess. If any signal gets forwarded incorrectly, the CLI might receive an interrupt before it can respond.

Fix: added `tini` as init, made it PID 1 so Python runs as a regular child process with normal signal semantics.

Result: no change. PID 1 wasn't the problem. Kept tini anyway since it's the right pattern.

### Actual root cause: SDK / CLI version mismatch

Isolated by running the bundled Claude CLI binary directly inside the container via `docker exec` instead of through the wrapper. The CLI itself was failing on roughly half of raw `--print` calls with "Execution error."

The wrapper ships with `claude-agent-sdk==0.1.18` and bundles a matching version of the Claude CLI binary. By mid-session, the current CLI was at v2.1.145. The newer CLI emits a `rate_limit_event` message in its stream-JSON output that `claude-agent-sdk 0.1.18` doesn't recognize — it raises an unknown message type exception, the wrapper catches it as "no response," and the request returns a 500.

Fix: upgrade the SDK. The wrapper's `pyproject.toml` declares `^0.1.18` (any 0.1.x), so installing `claude-agent-sdk==0.1.81` after `poetry install` in the Dockerfile stays within the declared compatibility range and overrides the pinned version:

```dockerfile
RUN pip install --no-cache-dir 'claude-agent-sdk==0.1.81'
```

And replace the bundled CLI binary with the current npm-installed version:

```dockerfile
RUN npm install -g @anthropic-ai/claude-code
RUN ln -sf $(which claude) /path/to/wrapper/bundled/claude
```

20/20 after that. 10/10 end-to-end through LiteLLM.

## End state

LiteLLM on closet routes Claude alias models through the closet bridge. The bazzite `claude-bridge.service` is disabled — unit file kept for rollback. Open WebUI on bazzite points at the closet LiteLLM. One AI router, multiple devices, billed against the subscription.

## The access-token refresh edge case

The bridge seeds its private `~/.claude/credentials.json` from OpenClaw's copy at container startup. When Anthropic rotates the token, the SDK refreshes it and writes the new value to the private copy. If the container restarts before the rotation propagates back to OpenClaw's copy, the entrypoint will overwrite the fresh token with a stale one. Worst case: one or two failed requests while the SDK does its own refresh. Worth monitoring for the first week.
