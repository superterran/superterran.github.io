---
title: "deploying a cloudflare worker with kv caching and github actions"
date: 2026-03-29
draft: false
categories: ["homelab"]
tags: ["cloudflare", "workers", "github-actions", "doughatcher", "ci-cd"]
summary: "Link preview service for doughatcher.com: Cloudflare Worker, KV cache, wrangler deploy via GitHub Actions. One YAML gotcha ate most of the debugging time."
---

The site needed a link preview service — fetch Open Graph data for URLs and cache the results so repeat lookups are fast. Cloudflare Workers was the right fit: runs at the edge, KV storage is built in, and the deploy story via wrangler is straightforward. The setup was mostly smooth except for one YAML issue that took longer than it should have.

## The worker

The worker receives a URL, checks the KV namespace for a cached preview, and returns it if found. On a cache miss it fetches the page, parses Open Graph tags, stores the result in KV, and returns it. Nothing unusual — standard Worker + KV pattern.

The worker is deployed via `wrangler deploy`. CI runs a second step after deploy to populate the KV cache from the existing URL backup: download the latest backup ZIP, scan it for Reddit URLs, prefetch previews for each one.

## The YAML bug

Two workflow files had the same issue: a Python multiline heredoc in a `run:` block with lines starting at column 0.

YAML block scalars are indentation-sensitive. A `run:` block's content is interpreted relative to the block's indentation level. Lines at column 0 end the scalar, so the Python code was getting silently truncated — the workflow file parsed without error, the job had zero steps, and ran with a 0-second duration. No obvious error message, just nothing happening.

Fix: replace the multiline heredoc with a single-line `python3 -c "..."`. Less readable but unambiguous to the YAML parser.

This is the kind of bug that's hard to spot because the YAML is syntactically valid. The file doesn't fail to parse — it just means something different than you wrote.

## Auth with wrangler 4

wrangler 4 accepts legacy global API key auth via `CLOUDFLARE_API_KEY` + `CLOUDFLARE_EMAIL` environment variables. The newer API token flow also works, but the global key was already available. Either approach sets the secrets in the repo and the workflow picks them up.

## package-lock.json

`setup-node` with `cache: 'npm'` requires a `package-lock.json`. The repo didn't have one. The cache step failed silently and fell back to a fresh install on every run. Running `npm install` locally and committing the lockfile fixed it.

## Thumbnail face-clipping

The preview card had a face-clipping issue — headshots were cropped from the middle, cutting off the top of the head. CSS fix: `object-position: center top` on the thumbnail image. Simple, but worth noting because it's easy to miss in a component that mostly renders fine.
