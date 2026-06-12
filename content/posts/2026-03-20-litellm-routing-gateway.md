---
title: "litellm as a local ai routing proxy"
date: 2026-03-20
draft: false
categories: ["homelab"]
tags: ["litellm", "bazzite", "ai-routing", "systemd", "ollama", "poe"]
summary: "LiteLLM sitting between the AI clients and the providers, routing by complexity tier. Simple prompts go local (Ollama), heavier work goes to subscription services."
---

The problem with having multiple AI providers is that every client needs to know about each one. LiteLLM solves this: one local OpenAI-compatible endpoint, all the routing logic in a config file, and any client that speaks the OpenAI API works against all providers at once.

The routing goal here: send trivial requests to local Ollama (free, GPU-accelerated, private), route heavier work to flat-rate subscription services (Poe covers Claude, GPT, Grok, Gemini at $17/month), and reserve the highest-capability models for when they're actually needed.

## Complexity-based routing

LiteLLM's complexity router assigns a complexity score to each request using a heuristic (prompt length, question complexity signals) and routes it to the appropriate model tier. No embedding API call, no latency overhead — pure heuristic, runs in-process.

Six tiers after a mid-session refactor based on actual Poe pricing:

| Tier | Model | Poe output $/1M | Use case |
|------|-------|-----------------|----------|
| nano | gpt-5-nano | $0.36 | Trivial lookups |
| fast | grok-4-fast | $0.50 | Quick questions, boilerplate |
| mid | gpt-4-mini | $1.40 | General coding |
| smart | gemini-2.5-pro | $7 | Hard problems, large context |
| max | claude-sonnet | $13 | Best quality coding/reasoning |
| apex | claude-opus | $21 | Hardest problems only |

The `auto` virtual model routes: SIMPLE → nano, MEDIUM → fast, COMPLEX → smart, REASONING → max. Fallback chain escalates tier on failure.

## One EnvironmentFile gotcha

systemd's `EnvironmentFile` does not support `export KEY=value` syntax — only `KEY=value`. The shell convention of `export` is not parsed here; it becomes a literal value including the word "export" as part of the key. Took a few minutes to notice why the proxy wasn't picking up the API keys.

## Local only, single worker

The proxy binds to localhost only — nothing accessible from the network. Multi-worker mode needs uvloop, which wasn't building on this machine (`gcc-12` missing, pre-built wheel available for a newer version). Single worker is fine for personal use; the proxy is fast enough that the worker count isn't the bottleneck.

## Anthropic API vs. Claude.ai Pro

One thing that clarified itself during setup: Anthropic API credits and a Claude.ai Pro subscription are separate billing systems. A Claude.ai Pro account has no API credits by default — API access requires purchasing credits on a separate Anthropic API account. Worth knowing before assuming Pro subscribers get API access.

## Result

Claude CLI, OpenCode, VS Code — all point at `http://localhost:4000/v1`. The config file picks the provider. Switching models or adding a new provider is a config change and a service restart, not a change in every client.
