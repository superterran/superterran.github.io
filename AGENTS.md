# AGENTS — superterran.net

Operational guide for any agent (currently Hobbs) working in this repo. **Read [`EDITORIAL.md`](./EDITORIAL.md) first — it's the contract.** This file is the *how*; that file is the *what.*

## Repo at a glance

- Hugo site, theme: **Blowfish** (submodule at `themes/blowfish`)
- Hosted on GitHub Pages, apex domain **superterran.net** (Cloudflare CNAME flatten → `superterran.github.io`)
- Built and deployed by `.github/workflows/deploy.yml` on push to `main`
- Local dev: `hugo server -D` (drafts shown) or `hugo server` (drafts hidden)

## Your job

You read Doug's session logs in `Context/sessions/` and dream-cycle output, find things worth writing about (per `EDITORIAL.md`), draft posts, and open PRs against this repo. Doug merges (for now). One post per PR, opened on a `post/` branch.

You do NOT:
- Merge your own PRs until Doug has flipped the graduation switch.
- Touch the theme submodule — if Blowfish needs an update, raise it in `#my-socials`.
- Modify the deploy workflow without flagging it explicitly in the PR description.
- Modify `EDITORIAL.md` without explicit Doug approval — it's the contract.

## Posting a draft

```bash
git checkout -b post/$(date +%Y-%m-%d)-short-slug
# create content/posts/YYYY-MM-DD-short-slug.md per EDITORIAL.md frontmatter
hugo --gc --minify  # local build sanity check
# verify nothing in the rendered HTML output looks wrong
gh pr create --title "post title" --body "$(cat <<EOF
Source: Context/sessions/YYYY-MM-DD-something.md
One sentence on what makes this post-worthy.

## Guardrail self-check

(answer YES/NO to each — all must be YES to open the PR)

- Avoids all hard-banned topics: YES/NO
- No real names from work / household / agents: YES/NO
- Title passes voice ban list: YES/NO
- Under 1000 words, or justified: YES/NO
- No CTA / wrap-up paragraph: YES/NO
- No content from confidential channels: YES/NO
EOF
)"
```

## Frontmatter shape (required)

```yaml
---
title: "lowercase fragmentary title"
date: YYYY-MM-DD
draft: false
categories: ["meta" | "homelab" | "mcp" | "infra" | "hugo" | "tools" | "ai" | "notes"]
tags: ["2-5", "specific", "tags"]
summary: "One concrete sentence."
---
```

## Local build verification

You can `ssh bazzite` and run Hugo locally if you need to verify rendering before opening a PR:

```bash
ssh bazzite "cd ~/repos/superterran.net && hugo --gc --minify"
```

If the build fails, fix it in your branch — don't open a PR that breaks the build.

## When things go wrong

- **Build fails on the Pages workflow:** flag in `#my-socials`, link the failed run, don't try to "fix forward" by force-pushing — open a fix PR.
- **Doug rejects a PR:** read his comment, internalize it, log it in your `memory/` for tone-tuning. Don't reopen the PR with a fix unless he asks for one; sometimes "rejected" means "not the right post," not "fix and retry."
- **Guardrail self-check returns a NO on something:** close the PR, kill the draft, move on. No rewriting around the rail.

## Related

- `EDITORIAL.md` — the contract (what does/doesn't go here, voice spec)
- `Context/agents/hobbs.md` (vault) — your canonical persona
- `workspace-hobbs/lanes/superterran-blog.md` — your operational lane spec
- `Context/blog-engine/manifest.md` — Sloane's editorial-loop targeting doughatcher.com (sibling system, different voice/audience)
