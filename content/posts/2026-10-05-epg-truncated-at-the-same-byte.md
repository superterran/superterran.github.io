---
title: "epg xml kept truncating at the same byte count"
date: 2026-10-05
draft: false
categories: ["homelab"]
tags: ["python", "xml", "docker", "debugging"]
summary: "A guide-data fetcher kept getting cut off at an identical byte offset on every run, and a line-based filter choked because the source emitted its whole XML as a single line."
---

An EPG (TV guide) fetcher I run started handing back truncated XML. Not randomly truncated — the exact same byte count, every single run. That consistency is the tell: it's not a flaky network, it's something structural cutting the read off at a fixed point.

Turned out the fetch was happening right after a separate call that also hit the same upstream, back to back. Whatever was serving the guide data couldn't keep the connection open that long under load, so it closed early at a predictable spot. Fix was boring: retry up to 4 times with a 30-second backoff between attempts, instead of treating one truncated read as the final answer.

That fixed the download, but the filtered guide was still coming back empty. The downstream step filters the XML line by line, and it turned out the source wasn't emitting one tag per line at all — the whole document was one line, no newlines anywhere. A line-based filter against a single giant line either matches everything or matches nothing, and it was matching nothing.

Fix there was to split on `><` while streaming the download in, so each tag lands on its own line before the filter ever sees it. After both fixes: guide data flowing, filtered ranges actually populated, no more silent empty results.

The pattern worth keeping: "truncated at the same spot every time" means look at what else is happening on that connection, not at your own parsing code. And never assume a text format is actually line-delimited just because it looks like it should be.
