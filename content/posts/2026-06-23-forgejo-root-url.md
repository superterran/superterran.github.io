---
title: "forgejo root_url before the tunnel exists"
date: 2026-06-23
draft: false
categories: ["homelab"]
tags: ["forgejo", "self-hosted", "cookie", "gotcha"]
summary: "Set root_url to the intended future domain before the edge tunnel was live. LAN login silently broke — session cookie scoped to the wrong host with Secure=true, rejected over HTTP."
---

Set up Forgejo LAN-only, planning to front it with a Cloudflare tunnel later. Configured `root_url` to the intended public hostname up front. The tunnel didn't exist yet — that was fine for now.

Login broke immediately. Form submits, page reloads back to the login screen, no error. Web UI came up, browsing worked, just no login.

Forgejo's admin dashboard showed a domain mismatch warning — configured `root_url` didn't match the request host. That looked like the diagnostic signal. It isn't, not fully.

`root_url` controls more than display: it scopes the session and CSRF cookies to that hostname and sets `Secure=true`. With `root_url = https://future-hostname/`, Forgejo issues session tokens scoped to that host. Requests coming in over LAN via HTTP get those cookies rejected by the browser — wrong origin, wrong scheme. Every POST silently fails.

Fix: set `root_url` to whatever you're actually hitting during setup. For LAN-only HTTP access, also add `COOKIE_SECURE = false`. When the tunnel is live and DNS resolves, swap `root_url` to the real domain and drop the override.

The domain mismatch warning is a notice about a discrepancy, not an explanation of the consequence. The admin warning and the login failure are related but the warning alone won't tell you why the form is bouncing you.
