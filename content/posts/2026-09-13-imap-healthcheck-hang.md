---
title: "the healthcheck was red for three weeks and nothing happened"
date: 2026-09-13
draft: false
categories: ["infra"]
tags: ["python", "imap", "docker", "debugging"]
summary: "A mail watcher hung mid-reconnect for three weeks because nothing ever crashed, so docker's restart policy never got a chance to fire."
---

`socket error: EOF`, then silence. That's the last line in the log before a mail watcher stopped delivering anything, three weeks ago. The container was still running the whole time — `restart: unless-stopped` never fired because the process never exited. It just stopped doing anything.

The healthcheck had been red the entire three weeks. Red healthchecks are supposed to be the thing that saves you here. This one couldn't, because a restart policy only cares about the process exiting, and a hung socket doesn't exit. It sits there, waiting on a read that's never coming back.

The cause: `imaplib.IMAP4_SSL(...)` constructed with no timeout. A half-open TCP connection — the kind you get from a mid-reconnect network blip — blocks the read forever. No timeout means no exception, no exception means no crash, no crash means no restart.

Fix was two parts. First, an actual timeout on the socket, so a hang turns into an `OSError` the existing reconnect logic already knows how to handle. Second, a watchdog thread that kills the process outright if the heartbeat goes stale for ten minutes, because the next wedge won't necessarily look like this one, and I'd rather trust "the process is dead" over "the process is doing something."

Restarted it. Backfill delivered in under a minute. The watchdog hasn't had to fire yet, but next time the wedge won't look like a socket EOF — it'll look like whatever novel way I haven't thought of, which is the whole reason it kills the process instead of trying to reason about the failure.
