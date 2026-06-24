---
title: "seeding a forgejo mirror from github"
date: 2026-06-24
draft: false
categories: ["homelab"]
tags: ["forgejo", "github", "self-hosted", "backup", "mirror"]
summary: "A shell script that enumerates all your GitHub repos and registers each as a Forgejo pull-mirror. Three gotchas: fine-grained PATs don't work across owners, big repos need a longer timeout, and empty repos need delete-and-recreate on retry."
---

Got Forgejo running on TrueNAS via the community catalog app. The web UI comes up, pull-sync defaults to 8h, everything works. What it doesn't do is seed anything — you start with zero repos.

The GitHub API returns your repos in pages. `gh repo list` handles pagination. For each repo, Forgejo's migrate API (`/api/v1/repos/migrate`) registers it as a pull-mirror when you pass `mirror: true`. That's the core loop.

Three things that bit me:

**Classic PAT, not fine-grained.** Fine-grained GitHub PATs are scoped per owner. I have a user account plus eight orgs — that's nine tokens to create and rotate. A classic PAT with `repo` scope covers all of them. Forgejo stores it per-mirror for resync; you don't touch auth again after seeding.

**migrate is synchronous.** The endpoint doesn't return until the clone finishes. Large repos — anything with a lot of history — can exceed 30 seconds. Bumped curl's timeout to 900s and it finished without issues.

**Empty repos on retry.** Run the script twice and Forgejo sees each repo already exists, refuses to recreate it. If the first run failed mid-clone, you get a repo that exists but is empty. Fix: on a collision, check whether the existing repo is empty; if so, delete it and re-register. A two-pass run converges cleanly.

End result: all repos pulling from GitHub on an 8-hour cycle, all browseable locally without touching GitHub.
