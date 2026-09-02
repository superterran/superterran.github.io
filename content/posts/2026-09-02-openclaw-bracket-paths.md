---
title: "array patches were too wide"
date: 2026-09-02
draft: false
categories: ["tools"]
tags: ["openclaw", "config", "cli", "json"]
summary: "OpenClaw's bracket-path config writes were the safer move once I realized array patches replace the whole list."
---

The command I wanted was not the big patch.

```sh
openclaw config set agents.list[3].model.fallbacks '[...]' --dry-run
```

That was the small discovery in the middle of wiring another model fallback into OpenClaw. I had a config array with several agents in it, and I needed to change one nested `fallbacks` list. The obvious-looking path was `openclaw config patch --file ...`, because I already had the replacement JSON in front of me.

That was the dangerous path.

Patch semantics replace arrays wholesale. If I patched `agents.list`, I had to re-supply every agent entry correctly. Same order, same inherited defaults, same per-agent weirdness. One typo in an unrelated agent and I would have turned a model-routing change into a config recovery session.

The CLI already had the better shape:

```sh
agents.list[3].model.fallbacks
```

Bracket paths work. `--dry-run` works. So the edit could stay exactly as small as the thing I meant to change: set this one nested array, preview it, validate it, then apply it.

I like this kind of tool affordance because it acknowledges the actual failure mode. Most config mistakes in these systems are not bad JSON. They are "I touched more surface area than I understood." Whole-array patches make that easy. Path writes make it harder.

There was another useful bit nearby: staging a patch file from inside the container with a heredoc avoided the old zero-byte stdin problem from `scp` / `docker cp`. Gross workaround, but reliable:

```sh
docker exec ... sh -c 'cat > /tmp/f <<EOF
...
EOF'
```

Still, the bracket path was the important part. It let the config write be as boring as the intent.
