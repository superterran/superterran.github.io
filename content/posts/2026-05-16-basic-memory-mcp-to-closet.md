---
title: "first mcp off bazzite, onto closet"
date: 2026-05-16
draft: false
categories: ["homelab"]
tags: ["closet", "truenas", "mcp", "basic-memory", "caddy", "docker", "migration"]
summary: "Moving basic-memory from a bazzite systemd user service to a docker-compose stack on closet. First service migration; establishes the pattern for the rest."
---

`basic-memory` was running as a systemd user service on bazzite-desktop — meaning if the workstation was off, the MCP server was gone. The closet TrueNAS box runs 24/7, so it's the right home for anything that should always be reachable. This is the first service migration. The goal was to get it working and nail down the pattern before doing the rest.

## Why docker-compose stacks, not TrueNAS Custom Apps

TrueNAS has its own Custom Apps interface for running containers. I'm not using it for these. The App catalog is fine for upstream software (cloudflared, jellyfin, openclaw itself), but for custom stacks it's clunky — you edit YAML through a web form, there's no clean way to version it in git, and the deploy workflow is opaque.

The approach instead: an `apps/` directory in the closet IaC repo, one subdirectory per service, each containing a `Dockerfile`, `docker-compose.yml`, and `Caddyfile`. A `bin/deploy-apps` script rsyncs the stack to the host and runs `docker compose up -d --build`. Fast iteration, version-controlled, deployable from anywhere on the LAN.

This pattern was in the CLAUDE.md for the repo as aspirational documentation. `apps/` was empty. This session is its first inhabitant.

## The two-port pattern

The container exposes two ports: a raw MCP port for LAN-direct access, and a Caddy-proxied port for the Cloudflare tunnel. Local clients (bazzite, opencode, VS Code) hit the LAN port directly — no auth token required, local network is trusted. The tunnel-facing port goes through Caddy, which validates a bearer token either as an `Authorization` header or a `?token=` query param.

This way the local dev experience stays frictionless and the public endpoint is gated without forcing every local client to carry credentials.

## UID 568

TrueNAS SCALE uses uid 568 for its app datasets. Running the containers as `568:568` matches the filesystem ownership convention, even though these aren't technically TrueNAS catalog apps. Worth noting: don't apply this to SMB-shared datasets — those use different UIDs and you'll break share access.

## ZFS and uv

uv defaults to reflink copy on Linux. ZFS on TrueNAS doesn't support reflinks and returns EAGAIN when uv tries one. uv retries with a regular copy and eventually succeeds, but logs a noisy wall of errors during the build. Setting `UV_LINK_MODE=copy` in the container environment silences it and goes straight to the working path.

## Re-indexing instead of copying the database

basic-memory stores its semantic index in a SQLite database with filesystem paths embedded. Copying that database to a new host with different mount paths would require editing those paths. With 45 notes and 25 KB of data, a full re-index from the markdown source files takes a few seconds. I wiped the database, let it re-build, and confirmed the first MCP call came back with real data.

## What moved, what stayed

The DNS record for the public endpoint already existed pointing at the wrong tunnel — some earlier partial migration attempt that never finished. Flipping it to the closet tunnel was a CNAME change.

The bazzite service is stopped and disabled, unit file kept for rollback. The local data on bazzite (`~/basic-memory/`) is now a frozen snapshot — safe to delete after a confidence interval.

The rest of the headless MCP services follow the same pattern. This session was about proving the pattern works.
