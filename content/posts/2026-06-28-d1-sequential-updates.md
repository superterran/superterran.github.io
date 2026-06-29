---
title: "d1 went silent at 900 sequential updates"
date: 2026-06-28
draft: false
categories: ["infra"]
tags: ["cloudflare", "d1", "workers", "gotcha"]
summary: "A reconcile Worker looped over ~900 rows and fired individual D1 UPDATE statements. D1 returned 500 with no useful body. Collapsed the loop into a single conditional UPDATE and it went through."
---

Writing a reconcile Worker: pull a set of IDs from one table, update a flag in another for each match. Standard enough pattern. Loop over the IDs, fire a prepared `UPDATE ... WHERE id = ?` for each one.

Got a 500 at runtime. D1 didn't surface a quota message, rate-limit header, or anything actionable — just a 500 from the Worker binding. Didn't fail in local dev (smaller dataset), didn't fail fast in prod. Worked fine for the first batch, fell over somewhere deep in the sequence.

The loop was firing roughly 900 sequential `.prepare().bind().run()` calls in a single Worker invocation. That was past whatever D1's implicit ceiling is for sequential Worker-bound statements.

Fix in this case: the condition was expressible in SQL directly. Instead of iterating and firing per-row UPDATEs, one statement covered everything: `UPDATE contacts SET is_member = 1 WHERE linked_id IS NOT NULL AND is_member = 0`. One trip, nothing to loop.

For cases where the rows to update come from a computed set and can't be reduced to a SQL predicate, D1's `batch()` API handles it — bundle the prepared statements into a single `.batch(statements)` call instead of individual awaits. Same number of statements, one round-trip, D1 handles it as a transaction and doesn't exhibit the same ceiling.

The threshold isn't documented. 900 was past it. Smaller reconciles on the same table were fine.
