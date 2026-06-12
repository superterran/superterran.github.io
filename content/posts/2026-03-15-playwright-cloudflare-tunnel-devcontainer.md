---
title: "e2e testing a local devcontainer through a cloudflare quick tunnel"
date: 2026-03-15
draft: false
categories: ["homelab"]
tags: ["playwright", "cloudflare", "devcontainer", "testing", "bazzite"]
summary: "Quick tunnels let Playwright MCP reach a local devcontainer from outside the network. Two gotchas: env.php system config silently overrides database config, and AJAX checkout steps need an explicit wait."
---

Playwright MCP can drive a browser against any URL — including a local devcontainer exposed through a Cloudflare quick tunnel. This avoids the complexity of running Playwright inside the container itself and lets the test session live in Claude's context window rather than buried in a test runner's output.

## Quick tunnels for devcontainer access

`cloudflared tunnel --url http://localhost:8443` gives a public HTTPS URL that forwards to the local devcontainer. The URL is temporary and changes on each run, but for an interactive test session it's fine — you update the application's base URL setting and run your tests.

Quick tunnels last longer than you'd expect. In practice: 12+ hours without expiring. For a long test session or overnight work, they hold.

## env.php overrides database config

This one is easy to miss. In Magento-family apps (and probably others that layer config the same way), there's a hierarchy:

1. `env.php` system config (highest priority)
2. `core_config_data` database table
3. Default values

If `env.php` has `system/default/web/unsecure/base_url` set, running `config:set web/unsecure/base_url https://new-tunnel.trycloudflare.com` appears to succeed — it writes to the database — but has no effect because `env.php` wins at runtime.

The symptom: you update the URL and the app keeps serving the old hostname. The fix is to update `env.php` directly, not the database config.

## Checkout AJAX timing

During checkout, address field changes trigger AJAX recalculations — shipping rates, totals, payment options. The shipping rate radio buttons become temporarily disabled during these recalculations. If Playwright clicks a radio button that's mid-recalculation, the click either fails or lands on a stale element.

Fix: after filling address fields, wait for the recalculation to complete before selecting a shipping method. `waitForSelector('.shipping-method:not([disabled])')` or a network idle wait both work.

## Payment methods aren't on by default

In a fresh dev environment, no payment methods may be configured. The checkout review step shows the payment section but with nothing to select. Enable a simple method via config before testing the full flow — Check/Money Order or Cash on Delivery are both good choices for local test environments since they have no external dependencies.

## Screenshots at every step

The agent can't see the browser. Screenshots are its only view into what's actually happening. Taking a screenshot after each significant action (navigate, form fill, click, page transition) means you can audit what happened after the fact and catch visual regressions that would be invisible in a text-only view.

Playwright MCP with `--save-trace` records a full session trace viewable at `trace.playwright.dev` — useful for post-session debugging without having to replay the test.
