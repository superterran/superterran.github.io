# Editorial guardrails — superterran.net

**This file is load-bearing.** It's the contract any agent (currently Hobbs) operating on this site is bound by. The same guardrails apply if I'm writing a post myself. If a draft is in violation, the PR doesn't ship — no exceptions, no "but this one is interesting."

## What goes here

- Personal projects: homelab, Bazzite, the closet TrueNAS box, MCP servers, agent infrastructure, Cloudflare tunnels, Hugo plumbing, Quadlets/systemd, devcontainers.
- Things I figured out the hard way. The gotcha that took 30 minutes is a better post than the architecture overview that took 2 hours to write.
- Tool tinkering, side-projects, dotfiles, scripts.
- Work-adjacent technical opinions where the opinion is the substance and no specific people/clients/projects identify themselves.
- Occasional reflections on tooling, AI/agents, the operator-vs-engineer life — as long as they're concrete, not motivational.

## What does NOT go here — hard bans

These are absolute. There's no judgement call, no "but it's interesting enough to be worth it." If a draft touches any of these, kill it.

1. **No real names from work.** Not clients, not coworkers, not the company. If a story is good but requires naming anyone I work with, the story doesn't run. The reader doesn't need the name to learn the lesson.
2. **No household / partner / family content.** Anything about Kaitlin, household planning, wedding planning, date nights, family dynamics — out. That's not this site's audience.
3. **No content from confidential channels.** Anything from `#dk-datenight`, anything from Kaitlin's shared channels, anything from any agent's private workspace, anything labeled surprise-mode anywhere — out, permanently.
4. **No wellbeing data.** Mood, weight, calorie, workout, sleep, anything from the wellbeing MCP — out. That's between me and the wellbeing agent.
5. **No active BA work, no client material, no engagement details.** The "I'm working on a thing for a Fortune-500 retailer" form is still out, even unnamed.
6. **No secrets, no tokens, no internal hostnames that aren't already public, no IP addresses for non-public services.** When in doubt, run `git secrets --scan` or equivalent before opening the PR. The published `*.superterran.net` services are fine; the `10.0.0.0/24` LAN side isn't.
7. **No security/CVE commentary** for current client engagements — that's the [experience-digest](https://experiencedigest.org) lane, not this one.

## Voice — what's banned, what's encouraged

The voice problem this site exists partly to avoid: hyper-smooth, headline-driven, AI-marketing-pamphlet feel. The other blog has its own voice for its own audience; this one is supposed to sound like a workshop notebook, not a content feed.

### Banned framings

- "How I built X" / "How I got Y working" — clickbait-formula titles.
- "N things I learned about Z" / "N lessons from Y" — listicle openers.
- "Here's the thing about…" / "Here's what nobody tells you about…"
- "The takeaway is…" or any wrap-up paragraph that re-states the thesis.
- "What this means for you" / "What this means for X shops."
- "Pro tip:" / "Hot take:" / "Unpopular opinion:"
- Alliterative headlines (banned outright — "Building Better Bazzite" type).
- "Want to know how I…" / "Curious about X? Read on" — CTAs.
- Tricolon cadence ("It's faster, simpler, and more reliable.") — too pat.
- Buzzwords: leverage, robust, seamless, unlock, paradigm, transformative, gamechanger, holy grail.
- The em-dash-between-clauses tic, especially as a setup-payoff (`X — and that's the whole story`). It's a known LLM tell.

### Encouraged

- Lowercase / sentence-case titles, often fragmentary. "why dreams-sync runs on closet" is a real title; "Why I Chose to Run Dreams Sync on Closet" is not.
- Lead with the artifact — the error message, the config snippet, the command, the screenshot. Not with a thesis.
- One observation per post. 200 words is fine. 100 is fine if 100 is all there is.
- Sentence-length variety. Some choppy, some long. Don't smooth it out.
- Doug's word choices, preserved. If he called it "the gross workaround," call it the gross workaround. Don't sand it to "the suboptimal interim solution."
- Quote the actual command/error verbatim. Don't paraphrase a stack trace into prose.
- Tangents allowed. "While I was in there I also noticed…" is fine.
- End where the thinking ends. No "in conclusion." No "if you have thoughts, let me know." Just stop.
- Sounding unsure is fine: "I don't know if this is the right call yet, but…" reads as human; certainty reads as marketing.

### Length

- Target 200–500 words.
- 100 is fine. Padding to look authoritative is worse than being brief.
- Never break 1000 unless there's a specific reason (a long config dump, a real step-by-step).

## Frontmatter conventions

```yaml
---
title: "lowercase / fragmentary title"
date: YYYY-MM-DD
draft: false
categories: ["one of: meta, homelab, mcp, infra, hugo, tools, ai, notes"]
tags: ["specific", "topic", "tags"]
summary: "One sentence. The first thing a reader sees in a card grid. Be concrete, not promotional."
---
```

- `categories`: pick ONE, not three. Multiple categories make navigation noisy.
- `tags`: 2–5 specific tags. Tags should be things you'd actually want to filter on later, not exhaustive metadata.
- `summary`: required. Treat it like the lede of a news article — concrete and specific. If your summary could fit a different post, rewrite it.

## PR workflow

1. Branch: `post/YYYY-MM-DD-short-slug` from `main`.
2. Add the post under `content/posts/YYYY-MM-DD-slug.md`. One post per PR.
3. PR title: same as the post title.
4. PR body: link to the source session log(s) the post is drawn from, plus a one-sentence note on what makes it post-worthy.
5. Run the guardrail self-check before opening the PR (see below).
6. Wait for Doug to merge. Don't auto-merge until the trust gate has been explicitly flipped (see graduation criteria, separate doc).

## Guardrail self-check

Before opening a PR, the agent must answer YES to all of these in the PR body. If any answer is NO or unclear, do not open the PR.

- Does the post avoid all hard-banned topics in the "What does NOT go here" section above?
- Are all proper nouns either (a) already public infrastructure I own, (b) widely-known tools/products, or (c) my own first name? No client names, no coworker names, no household names, no agent confidants.
- Does the title pass the voice ban list? (No alliteration, no "How I", no listicle opener.)
- Is the post under 1000 words, or if longer, is there a specific reason it has to be?
- Does the post end without a CTA or wrap-up paragraph?
- Is there any content that came from a confidential channel (datenight, wellbeing, household, BA work)? If yes, the post does not ship — no rewrite will fix this.

## When the rules are unclear

If a draft is near a line, the call is **don't ship**. The cost of holding a borderline post is zero; the cost of one wrong call here can be unbounded. Hold and ask in `#my-socials`.
