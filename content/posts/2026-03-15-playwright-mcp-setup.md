---
title: "playwright mcp on bazzite: persistent profiles and a browser version gotcha"
date: 2026-03-15
draft: false
categories: ["homelab"]
tags: ["bazzite", "playwright", "mcp", "browser-automation", "ai-tooling"]
summary: "Microsoft's official Playwright MCP server wired into Claude and OpenCode. One tricky bit: the MCP package bundles its own playwright-core, so you need to install Chromium via the package's binary, not the globally installed playwright."
---

`@playwright/mcp` (Microsoft's official package, 28k+ stars) gives Claude and OpenCode a set of browser-control tools: navigate, click, fill forms, take screenshots, capture accessibility snapshots, run assertions. It's the most fully-featured option and gets day-of-release updates alongside Playwright itself.

## Installation

```bash
npm install -g @playwright/mcp
```

Then install Chromium using the MCP package's own playwright binary, not the globally installed one:

```bash
/path/to/@playwright/mcp/node_modules/.bin/playwright install chromium
```

This is the main gotcha. The MCP package bundles `playwright-core` at a specific version that may differ from whatever `playwright` you have globally. Each playwright-core version expects its own Chromium build. If you install Chromium via the global `playwright install`, it installs the wrong version for the MCP package's bundled core and the browser won't launch.

## MCP server config

For Claude CLI:
```bash
claude mcp add playwright -s user -- npx @playwright/mcp/cli --headless --caps vision --save-trace
```

Key flags:
- `--headless` — runs without a visible window; right for Gamescope/systemd contexts where there's no display manager to render to
- `--caps vision` — enables screenshot tools; without this the agent can see the DOM but not the rendered page
- `--save-trace` — records a full session trace viewable at trace.playwright.dev; useful for debugging what the agent actually did
- `--ignore-https-errors` — needed for local dev with self-signed certs

## Persistent profiles

Login state persists across MCP sessions in a profile directory (`~/.cache/ms-playwright/mcp-chromium-profile/`). Once you log into a site through the MCP, you stay logged in. This is useful in practice — you don't re-authenticate every chat session.

The flip side: if you want a clean slate, you need to clear the profile directory explicitly.

## Bazzite-specific notes

`playwright install-deps` tries to run `apt-get` to install system dependencies. On Bazzite (Fedora Silverblue, immutable rootfs) this fails. It doesn't matter — the system libraries that Chromium needs are present from the rpm-ostree base. The browser launches fine without the dependency install step.

The MCP server runs headless, which works correctly in the Gamescope desktop environment. There's no conflict with Gamescope owning the display because the MCP browser isn't trying to open a window on any physical output.

## What the agent can actually do

With `--caps vision` enabled, the agent has screenshot capture plus accessibility snapshot tools. The accessibility snapshot is often more useful than a screenshot for navigation — it gives a structured DOM view that's cheaper to include in context and doesn't require vision capabilities. Screenshots are for visual inspection and confirmation.

The practical workflow: navigate to a page, take an accessibility snapshot to understand structure, interact via click/fill/select, take a screenshot to confirm the result.
