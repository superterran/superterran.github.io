# superterran.net

A working notebook. Personal projects, infra, things I'm tinkering with.

- **Live:** [superterran.net](https://superterran.net)
- **Stack:** Hugo + Blowfish theme, GitHub Pages, Cloudflare DNS at apex
- **Sibling site:** [doughatcher.com](https://doughatcher.com) (positioned for Adobe Commerce / public voice; this site is the workshop)

## How it works

Posts are drafted by Hobbs (one of my [OpenClaw](https://openclaw.ai) agents) reading my session logs, opened as PRs against `main`, and merged by me. See [`EDITORIAL.md`](./EDITORIAL.md) for the editorial guardrails — they're load-bearing.

## Local dev

```bash
git clone --recurse-submodules https://github.com/superterran/superterran.net.git
cd superterran.net
hugo server -D
```

Then open http://localhost:1313.

## Layout

```
config/_default/   Hugo config (split per Blowfish docs)
content/           Posts and pages
themes/blowfish/   Theme submodule
static/            CNAME, favicons, static assets
.github/workflows/ Pages deploy
EDITORIAL.md       Guardrails — read before contributing
AGENTS.md          Operational guide for the agent posting here
```

## License

Content: © Doug Hatcher, all rights reserved.
Theme: Blowfish, MIT.
