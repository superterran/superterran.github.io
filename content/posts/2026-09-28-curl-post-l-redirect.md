---
title: "curl -X POST -L is a trap"
date: 2026-09-28
draft: false
categories: ["tools"]
tags: ["curl", "http", "debugging", "redirects"]
summary: "curl -X POST -L keeps POST across 302s. Real HTTP clients don't."
---

Spent 20 minutes chasing a 404 that wasn't there. An endpoint was POST-only — GET returned 404. The real browser flow worked: POST the form, get a 302, follow it with GET, land on the success page.

So I curl'd it:

```
curl -X POST -L https://example.com/subscribe
```

404.

Checked the endpoint three different ways. Checked the payload. Checked headers. Everything looked right. The 404 was real and reproducible.

It wasn't.

Here's the thing: `curl -X` sets a *custom method*. It overrides curl's default behavior. Normally, when curl follows a redirect (`-L`), it changes the method based on the status code. 301/302 → GET. 307/308 → keep the method. That's what browsers do, that's what `fetch()` does, that's what every HTTP client does.

But `-X POST` bypasses that logic. Curl sees "user said POST" and posts through every redirect, including the 302. So:

```
POST /subscribe → 302 → POST /check → 404
```

The `/check` endpoint expected GET. It got POST. 404.

`-X` doesn't mean "use POST for the first request." It means "use POST for every request in this chain."

The fix: drop `-X` and use `--data` or `-F` instead. When curl sees a data payload, it uses POST by default for the first hop — but will follow redirect semantics normally after that. Or, if you're just probing, use `--post301` / `--post302` / `--post303` to be explicit about which redirects should keep the method.

I'd used `curl -X POST -L` for years without hitting this because most POST endpoints either don't redirect or the redirect target accepts both methods. The day you hit a POST-only endpoint that 302s to a GET-only page, it bites.

This is documented in the man page. It's just not the part anyone reads.