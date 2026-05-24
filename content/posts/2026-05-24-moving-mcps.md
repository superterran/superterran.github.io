---
title: "moving mcps off the gaming machine"
date: 2026-05-24
draft: false
categories: ["homelab"]
tags: ["truenas", "mcp", "docker", "migration"]
summary: "basic-memory, mcpvault, wellbeing, and Open WebUI moved from the bazzite workstation to the closet NAS in mid-May. The VRAM recovery was the real payoff."
---

Three MCP servers and Open WebUI moved from the gaming workstation to the closet NAS in mid-May. The decision wasn't complicated: those services don't need a GPU, and a machine that's sometimes gaming isn't a good host for always-on infrastructure.

The migration pattern was the same for each:

1. Write a Docker Compose stack for the NAS in the closet IaC repo.
2. Add an auth proxy in front of the service.
3. Add a Cloudflare tunnel entry to reach it externally.
4. Update the MCP client configs (OpenCode, VS Code, Claude Code) to point at the new address.
5. Disable the old unit on the workstation.

Each one took about an hour. Most of that time was getting the tunnel ingress right the first time.

The VRAM improvement on the workstation was better than expected. Before the migration, several Docker containers were running alongside Ollama, Kokoro TTS, and Speaches STT — not using GPU directly, but pulling memory and CPU from the same machine that was trying to serve those models. Getting those containers off made Ollama's cold starts noticeably faster, and TTS/STT latency dropped.

The constraint that didn't move: `task-dispatch`, the MCP server that dispatches Claude Code as subprocesses. It depends on the workstation's local environment and can't usefully run on a low-power NAS. That one stays.

The broader principle: the NAS handles coordination and data; the workstation handles compute. Anything that doesn't fit that split is a sign something is in the wrong place.
