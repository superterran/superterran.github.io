---
title: "nine stacks, zero commits"
date: 2026-06-12
draft: false
categories: ["homelab"]
tags: ["closet", "truenas", "iac", "git", "drift"]
summary: "The same audit that caught the thinkingDefault bug found nine app stacks running on closet with no corresponding git state. They'd been live for months."
---

Same audit run that found [the reasoning wasn't running](/posts/2026-06-12-reasoning-wasnt-running/). Pulled up the closet repo while I was there.

Nine docker-compose stacks in production, running fine. Zero of them tracked in git.

They'd accumulated across a few months of iterative deploys — stand up a container, tweak it until it works, move on. The compose files were on the host. The IaC repo had the original layout. The gap between them had grown silently while everything kept running.

Same pattern on bazzite: a batch of June config changes were live in the system but uncommitted. Found them by running `git status` in the repo, not because anything was broken.

Neither of these fails loudly either. The stacks run. The bazzite config applies. The system behaves exactly as it should, because the filesystem is the truth — git is supposed to be a copy of it, not the other way around. The divergence only surfaces when you go looking.

The fix was two commits: one for closet (nine stacks added, d092606), one for bazzite (12f2970). Two `git add -A && git commit` runs, no rewrites. The work was already done; this was just capturing it.

The harder part is the habit. A deploy that ends without a commit is a deploy that will eventually be invisible. I've set up enough IaC tooling to know this is obvious — and still managed to let it drift for months on personal infra where no one else is looking.
