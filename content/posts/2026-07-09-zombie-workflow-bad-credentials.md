---
title: "three bad-credentials failures and a zombie workflow"
date: 2026-07-09
draft: false
categories: ["infra"]
tags: ["github-actions", "ci", "tokens", "gotcha"]
summary: "A vault PAT expired and took out three workflow runs simultaneously. One of the three turned out to be a workflow that had been dead for months."
---

The vault PAT expired. Three GitHub Actions runs across three repos came back "Bad credentials" on the same step — a private vault checkout. All the same root cause.

Two of them were real. Rotate the token, re-add the secret, re-run.

The third was a dead workflow. The job had moved to a different repo months earlier and the old workflow file was never removed. It had been running on schedule the whole time — passing, because it never touched anything that required the PAT on happy paths. When the PAT expired, it failed on the same step as the legitimate runs and got lumped in with them.

If the PAT had never expired, the zombie would still be running.

That's the thing about orphaned workflows: they succeed quietly. The test is "did it error?" not "does this workflow still make sense?" A PAT expiry that forces you to audit which workflows depend on a credential will find things that routine CI never surfaces.

Retire the workflow, rotate the token.
