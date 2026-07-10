---
title: "micro.blog's hugo version is 0.91.2"
date: 2026-07-10
draft: false
categories: ["hugo"]
tags: ["micro.blog", "hugo", "gotcha", "templates"]
summary: "Micro.blog builds themes on Hugo 0.91.2. A template function that works locally on 0.147 can break silently — one page stays frozen while the rest of the deploy goes through."
---

Micro.blog builds themes on Hugo 0.91.2. If you develop locally on a modern Hugo (0.147 as of now), that gap is usually harmless — most template functions work the same.

When it isn't: Micro.blog doesn't fail a deploy on a per-page template error. It keeps the last-good HTML for the broken route and rebuilds everything else. The failed page sits frozen with a stale `last-modified` timestamp. Everything around it updates normally.

The symptom looks like a caching issue. One route isn't refreshing; hit purge a few times; nothing changes. Meanwhile, CSS changes (applied on theme reload, separate from the page build) go live fine. That makes it look like the HTML is stuck but the assets are okay, which sends you hunting in the wrong direction.

`strings.TrimSpace` was the culprit in my case. The `strings` namespace functions aren't available in 0.91.2; the local build works fine, Micro.blog errors silently on that one page.

Fix I'm keeping: a second Hugo binary at `/tmp/hugo0912/hugo` — extended, v0.91.2, built from the release archive. Run a test build against it before pushing any Micro.blog theme change. Doesn't need to be in PATH; just something to run `./hugo build` against as a pre-push sanity check.
