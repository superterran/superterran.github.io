---
title: "building a personal tracking mcp server with fastmcp"
date: 2026-04-05
draft: false
categories: ["homelab"]
tags: ["mcp", "fastmcp", "python", "systemd", "sqlite", "health-tracking"]
summary: "FastMCP makes it straightforward to build a personal data MCP server. Key patterns: separate bridge vs. server, SQLite-backed markdown snapshots, and the systemd EnvironmentFile export gotcha."
---

FastMCP (Python) handles the MCP server layer cleanly — you write annotated functions, decorate them with `@mcp.tool()`, and the framework handles JSON-RPC, schema generation, and transport. This is the pattern for a personal tracking server: SQLite for storage, a background resource for ambient context, and a separate HTTP bridge for external data ingestion.

## Separate bridge vs. MCP server

Two services, not one:

- The **MCP server** is spawn-on-demand. Claude or another client launches it, uses it, exits. That's fine for queries and updates initiated by the user.
- The **HTTP bridge** needs to be always-on. If it's waiting to receive data from an external source (an iOS app, a webhook, an API push), it can't be spawn-on-demand — the data arrives when it arrives, not when a client session is open.

The bridge receives incoming data and writes to the shared SQLite database. The MCP server reads from the same database. Clean separation: no shared process, no resource contention, independent restart behavior.

## The MCP resource for ambient context

FastMCP supports `@mcp.resource()` — a URI-addressable resource that gets included in the conversation context automatically when configured, without requiring an explicit tool call. This is useful for a daily snapshot: the current state of whatever you're tracking gets pulled into context at the start of every session.

The resource builds a readable text summary from the database state — today's entries, running totals, recent trends. Text, not JSON — because it's going into a language model's context and should be readable prose.

## SQLite as the storage layer

One database file, standard CRUD. All the entries are insert-only (log entries, not updates), so there's no concurrency issue between the bridge and the server even when they're both running. The bridge writes; the server reads. No locking needed.

## FastMCP's `run_http_async` for remote access

For HTTP/SSE transport (remote Claude instances, Open WebUI), FastMCP 3.2 uses `run_http_async()` with `transport='streamable-http'`. The API changed between major versions — the older `run()` with a transport kwarg doesn't work in 3.x. Worth checking the FastMCP changelog if you're upgrading from an earlier version.

## systemd EnvironmentFile: no `export`

systemd's `EnvironmentFile` directive parses `KEY=value` format. Shell convention is `export KEY=value` — the `export` keyword is not parsed and ends up treated as part of the key name. The result is that the environment variable is never set, with no error message.

```ini
# works
EnvironmentFile=/path/to/.env

# .env contents (correct)
MY_API_KEY=abc123

# .env contents (wrong — export keyword silently breaks it)
export MY_API_KEY=abc123
```

If you have a `~/.env` that you source interactively (where `export` is valid syntax), it won't work as an `EnvironmentFile` without modification. Keep separate files for interactive use and systemd, or use the `key=value` format everywhere.

## FastMCP callable-factory pattern

When FastMCP tools need access to a shared resource (like a database connection or a config object), the callable-factory pattern works cleanly: a factory function creates a callable with the shared state closed over it, and that callable is passed to `mcp.add_tool()`. This sidesteps the need for global state while keeping each tool function testable in isolation. The pattern was stabilized in FastMCP 3.x after some API churn in earlier versions — if you see decorator-based tools not binding correctly, checking whether you're closing over a connection or passing it explicitly is a good first debug step.
