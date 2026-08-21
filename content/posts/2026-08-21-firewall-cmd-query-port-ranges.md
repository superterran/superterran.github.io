---
title: "firewall-cmd --query-port doesn't see inside ranges"
date: 2026-08-21
draft: false
categories: ["homelab"]
tags: ["firewalld", "firewall-cmd", "ports", "gotcha"]
summary: "Querying a specific port number that falls inside an allowed range returns 'no'. firewall-cmd --query-port does exact string matching, not range containment."
---

Hardening run on the closet box. Spot-checking ports that should be closed. Ran this:

```
firewall-cmd --query-port=8080/tcp
```

Got `no`. That's the answer I wanted — confirmed the port is closed.

Except it wasn't.

A different part of the config had added `8080-8090/tcp` as a range earlier. Port 8080 was very much accessible. But `--query-port=8080/tcp` does an exact string match against the port list — it looks for `"8080/tcp"` and doesn't find it, because what's in the list is `"8080-8090/tcp"`.

```
firewall-cmd --list-ports
# → 8080-8090/tcp ...
```

Both commands agree with each other. Neither is lying. `--list-ports` shows the range. `--query-port=8080/tcp` checks for a literal match, finds none, returns `no`. They're operating on different abstractions.

For verification during a hardening pass, `--list-ports` and actual connect tests are more useful than per-port queries. `--query-port` is checking your configuration text, not whether the port is actually reachable.
