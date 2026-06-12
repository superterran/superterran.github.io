---
title: "building an mcp server for a laravel devcontainer"
date: 2026-03-14
draft: false
categories: ["homelab"]
tags: ["laravel", "mcp", "devcontainer", "php", "ai-tooling"]
summary: "11 tools, 1 resource, STDIO transport inside a devcontainer. php-mcp/laravel does the heavy lifting; the main gotcha is pcntl."
---

`php-mcp/laravel` makes it straightforward to expose a Laravel application as an MCP server — AI assistants can then query routes, run Artisan commands, inspect database schema, and tail logs directly from the chat context. The server runs over STDIO transport inside the devcontainer, which keeps it scoped to the project without exposing any network ports.

## What's exposed

Six tool classes, attribute-based discovery:

- **Routes** — list routes with optional method/path filters
- **Config** — read config values; `laravel://environment` resource for the current environment
- **Artisan** — run commands from a safety allowlist, list available commands
- **Database** — SELECT-only queries (100-row max), list tables, describe table schema
- **Models** — list Eloquent models, inspect relationships via reflection
- **Logs** — tail with level filtering, clear logs

The Artisan allowlist prevents destructive commands (`migrate:fresh`, `db:wipe`, etc.) from being callable. The database tool is SELECT-only by construction — no update path.

## pcntl is not in the base image

The STDIO transport requires the `pcntl` PHP extension. The standard `mcr.microsoft.com/devcontainers/php` image doesn't include it. Add it to the Dockerfile:

```dockerfile
RUN docker-php-ext-install pcntl
```

Without this, the server starts but the STDIO loop doesn't work. The error isn't always obvious.

## How STDIO transport works with devcontainers

The MCP client (OpenCode, Claude CLI, etc.) needs to spawn the server process and communicate via stdin/stdout. With a devcontainer, the command is:

```bash
devcontainer exec --workspace-folder /path/to/project php artisan mcp:serve
```

`devcontainer exec` properly forwards stdin/stdout for interactive communication. `docker exec` also works if the container is already running.

One behavioral detail: STDIO MCP servers need stdin to stay open. If you pipe a single message and close stdin, the server exits before it can respond. This is expected for STDIO servers — the client holds the connection open for the session.

## Xdebug noise is harmless

The devcontainer has Xdebug installed. When the MCP server starts, Xdebug writes "Could not connect to debugging client" to stderr. This doesn't affect the STDIO transport since the MCP client only reads stdout. Worth knowing if you see it in logs and wonder if something's broken.

## Project-scoped config

The MCP server only loads when the client opens from this specific project directory. In OpenCode this is a project-level `opencode.json` rather than the global config. Scoping prevents the server from loading in unrelated projects where it would be useless and potentially confusing.
