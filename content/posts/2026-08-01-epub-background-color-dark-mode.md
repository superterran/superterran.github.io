---
title: "epub background-color broke dark mode"
date: 2026-08-01
draft: false
categories: ["tools"]
tags: ["epub", "publishing", "css", "dark-mode"]
summary: "Setting background-color: #fff in an EPUB stylesheet renders as a grey slab in dark mode. The fix was removing the declaration entirely."
---

Building an EPUB stylesheet. Dark mode looked wrong — background wasn't going dark, just a flat grey.

Added `background-color: #fff` to body explicitly. Reasoning: if the rendering is ambiguous, lock it to white. That made it visibly worse. Grey slab got more opaque, text contrast degraded.

Fix: remove the color declaration entirely. EPUB readers implement dark mode by overriding or inverting author stylesheets. When you declare `background-color: #fff`, you're locking the renderer out — it can only override what you haven't declared. Remove it and the renderer does the right thing.

Same session, different problem: `nav[epub|type=toc]` without an `@namespace` declaration is invalid CSS. An invalid selector in a comma-separated list silently kills the whole rule. So this:

```css
nav[epub|type=toc], a { color: inherit; }
```

…was silently discarding `color: inherit` for `a` too. Link color was wrong and there was nothing obvious to debug — the declaration just wasn't being applied at all. Split the selectors.
