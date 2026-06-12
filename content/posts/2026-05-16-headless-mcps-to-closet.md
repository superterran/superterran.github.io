---
title: "migrating the rest of the mcp stack to closet"
date: 2026-05-16
draft: false
categories: ["homelab"]
tags: ["closet", "truenas", "mcp", "mcpvault", "wellbeing", "obsidian", "cloudflare", "docker", "migration"]
summary: "After basic-memory, the rest: mcpvault, wellbeing, and an Obsidian Sync replica all move off bazzite-desktop and onto closet in one session. Bazzite goes back to GPU work."
---

The [basic-memory migration](/posts/2026-05-16-basic-memory-mcp-to-closet/) proved the pattern. This session ran the rest of the headless MCP stack through it: mcpvault (vault read/write), wellbeing (health tracking), and an Obsidian Sync replica so closet has a local copy of the vault to serve from.

Bazzite-desktop is now back to what it should be doing: GPU work and interactive sessions. Everything that should run 24/7 runs on closet.

## Obsidian Sync as the vault replication mechanism

The tempting approach for getting the vault onto closet would be an SMB or NFS mount back to bazzite. That introduces a mount layer to maintain and `.sync.lock` contention between the desktop Obsidian app and whatever's reading remotely.

The better approach: add closet as a second Obsidian Sync peer. Obsidian Sync is already paid for and handles E2EE replication. Closet gets its own device identity, writes via MCP tools land on closet's replica, and Obsidian Sync propagates them back to bazzite where the desktop app picks them up. No mount layer.

The login step was awkward. `ob login` interactive mode froze during the password/MFA step — suspected TTY race inside the container. The fix was copying bazzite's `auth_token` file directly to the closet config volume (same Obsidian account), then running `ob sync-setup --device-name closet` to register closet as its own sync peer. One interactive step for the E2EE password, then it bootstrapped.

## mcpvault on closet

mcpvault wraps the vault as an MCP server. The closet version runs against the local Obsidian Sync replica — so it reads the vault directly from disk without touching bazzite. The bridge code from the original SSE bridge got a small generalization: hardcoded `127.0.0.1` and `HOME` paths replaced with environment variables so it works from inside a container.

## wellbeing on closet

The wellbeing MCP's source code hardcodes `~/Documents/Cloud Vault` as the vault path. Rather than fork the repo, the Dockerfile adds a filesystem symlink inside the container: `/state/Documents/Cloud Vault → /vault`. The container sees the expected path, the actual vault data comes from the Obsidian Sync replica on closet. One Dockerfile line, no source fork to maintain.

Two compose services — `wellbeing` and `wellbeing-bridge` — share the same image build. Different entrypoints, same data.

## Cloudflare tunnel drift

Discovered mid-session that the tunnel ingress config gets silently overwritten. OpenClaw or the cloudflared catalog app occasionally PUT a fresh ingress config, losing manually-applied rules. Found this by comparing two captures taken a few minutes apart: an ingress entry added earlier in the session was gone.

The fix: capture the desired state as JSON in the IaC repo (`inventory/cloudflare-tunnel.json`), with an `apply-cloudflare` script that diffs live state against inventory and PUTs inventory as desired state when they diverge. Any future drift is now a one-command recovery.

## What didn't move

`task-dispatch-mcp` stays on bazzite. Its job is spawning Claude CLI processes on the local machine. Moving it to closet would mean either SSHing back to bazzite for each dispatch (defeating the purpose) or running Claude on closet (which doesn't have the GPU). Some services belong on the machine they're meant to operate.

## End state

Five docker-compose stacks in `closet/apps/`, all deployed via `just deploy-apps`. Local clients point at closet's LAN address. Public tunnel endpoints show connected. Bazzite's headless MCP services stopped and disabled, unit files kept for rollback.
