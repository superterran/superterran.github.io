---
title: "detector evasion was the wrong goal"
date: 2026-07-30
draft: false
categories: ["ai"]
tags: ["mcp", "voice", "llm", "writing"]
summary: "Started with the standard anti-slop toolkit — perplexity, GLTR fingerprinting, detector evasion. A 61% false-positive rate on human writing changed the frame."
---

Designing a voice engine. Original goal: make generated text harder to detect as generated. Standard approach — flatten perplexity curves, target GLTR fingerprints, keep token surprisal in the human distribution.

Then: Liang et al. (arXiv:2304.02819). Perplexity-based AI detectors have a 61% false positive rate on human text written by non-native English speakers. Sixty-one percent. Worse than a coin flip against a non-trivial user population. Optimizing against a broken signal is busywork.

The flatness critic got demoted from gate to optional note.

Reframe: the goal isn't to seem human. The goal is to write better. Those turn out to need completely different tooling.

New design: two tools. `get_voice_context` runs before you write — injects few-shot examples from a per-surface writer palette plus a fingerprint of your own best work. `critique_draft` runs after — scores against a missing-moves rubric, runs the draft against an empirical cliché corpus (berenslab/llm-excess-vocab), flags where the text went flat.

The per-surface part matters more than I expected. Different publishing surfaces pull toward different stylistic models. The tool needs to know which surface it's writing for, not just "write well."

Tucker Max was tried as a palette exemplar and cut fast. Fratire voice doesn't compose with analytical clarity — the mix reads as tonal whiplash, not texture.

Still some work: corpus assembly, building the surface fingerprints, hosting decision. But the design is cleaner since dropping evasion as the optimization target. You stop asking "would a detector flag this" and start asking "does this sound like the writers I want to be when I'm not lazy."
