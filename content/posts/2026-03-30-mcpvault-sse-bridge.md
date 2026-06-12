---
title: "exposing a stdio mcp server over http/sse"
date: 2026-03-30
draft: false
categories: ["homelab"]
tags: ["mcp", "mcpvault", "cloudflare", "systemd", "bazzite", "obsidian"]
summary: "mcpvault runs over stdio. Claude.ai wants HTTP SSE. A small Node bridge spans the gap — spawn-per-session, bearer auth, Cloudflare tunnel."
---

mcpvault is an MCP server that gives Claude read/write access to an Obsidian vault. It runs over stdio — you pass it to Claude Code as a local process and it works great. The problem is that `claude.ai` and remote clients can only connect to HTTP/SSE endpoints, not local stdio processes. The vault lives on the local machine; the remote clients can't reach it directly.

The fix is a bridge: a small server that accepts HTTP/SSE connections, spawns an mcpvault process per session, and translates between the two protocols.

## How it works

The bridge listens on a local port. On each incoming connection it spawns `mcpvault` as a child process and pipes the SSE stream to its stdin/stdout. The MCP session is scoped to that connection — when the client disconnects, the child process exits.

Auth is a bearer token checked on every connection, accepted either as an `Authorization: Bearer ...` header or a `?token=...` query parameter. The query param form is useful for Claude Code's `claude mcp add --transport http` flag, which constructs the URL as a string.

The token lives in a `.env` file with `chmod 600`. Not in git.

## Systemd user service

The bridge runs as a systemd user service so it starts with the login session and restarts on failure:

```ini
[Unit]
Description=mcpvault remote SSE bridge

[Service]
EnvironmentFile=%h/repos/mcpvault-remote/.env
ExecStart=node %h/repos/mcpvault-remote/server.js
Restart=on-failure

[Install]
WantedBy=default.target
```

`WantedBy=default.target` with `loginctl enable-linger` keeps it running even when no interactive session is active.

## Cloudflare tunnel ingress

The bridge is reachable externally via the desktop's existing Cloudflare tunnel — the same tunnel that serves other local subdomains. Adding a new hostname is a config entry in the tunnel's ingress rules pointing at `localhost:<port>`. One DNS CNAME pointing at the tunnel, one ingress rule, done.

## The pattern

Any stdio MCP server can be wrapped this way. The bridge code is generic — it doesn't know anything about mcpvault specifically, just the MCP wire protocol over stdio. The same approach works for any server that doesn't natively support HTTP transport.

The main trade-off: spawn-per-session means a small startup cost on each connection. For interactive use it's imperceptible. If you needed high-frequency connections from multiple clients simultaneously, you'd want a long-lived process instead — but for personal use the per-session model is simpler and avoids shared state between sessions.
