---
title: "eight days of red ci for a route that didn't exist yet"
date: 2026-07-13
draft: false
categories: ["infra"]
tags: ["github-actions", "cloudflare-workers", "ci", "shell"]
summary: "The deploy step succeeded every time. The verify step was red. The health-check URL had never been provisioned."
---

The deploy workflow for a Cloudflare Worker had been reporting failure on every run since I first set it up. Eight days. The worker was updating fine — I could see new code in the responses — but CI was consistently red.

The Verify step looked like this:

```yaml
- name: Verify
  run: |
    CODE=$(curl -s -o /dev/null -w "%{http_code}" https://mcp.example.com/)
    echo "status: $CODE"
    if [ "$CODE" != "200" ]; then exit 1; fi
```

`curl` was returning exit code 6: could not resolve host. `mcp.example.com` was the planned custom domain, but the worker was configured `workers_dev = true` with the routes block commented out. The custom domain was never provisioned. The real endpoint was on `*.workers.dev`.

The catch: `run:` blocks execute under `set -e`. A bare command substitution `$(curl ...)` that exits nonzero aborts the shell immediately, before it reaches the `echo` or the `if` block. The conditional was never evaluated. CI was red because of the unresolved hostname, full stop.

The fix:

```yaml
CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 https://mcp.example.com/ || echo "000")
```

`|| echo "000"` absorbs the curl failure and keeps the shell alive. `--max-time 10` keeps it from hanging on a slow DNS timeout. Now the step prints `mcp.example.com → 000` (tolerated) and continues on to check the live workers.dev URL.

The `set -e` / command-substitution interaction is worth keeping in mind: `x=$(thing)` failing is not the same as `thing || true`. The substitution form propagates the exit code to the shell when running under `set -e`, but only if the assignment itself is a simple statement — some shells handle this differently. When in doubt, add the `|| echo fallback`.
