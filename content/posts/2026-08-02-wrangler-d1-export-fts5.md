---
title: "wrangler d1 export chokes on FTS5"
date: 2026-08-02
draft: false
categories: ["infra"]
tags: ["cloudflare", "d1", "wrangler", "fts5", "gotcha"]
summary: "Adding a full-text search index to a D1 database breaks wrangler d1 export. The fix is to enumerate user tables from sqlite_master and export them individually."
---

Had a backup job using `wrangler d1 export`. Ran fine until I added a full-text search index. Next scheduled run:

```
Error: cannot export databases with Virtual Tables (fts5)
```

FTS5 is SQLite's full-text search extension. Creating a virtual table with FTS5 also creates several shadow tables the extension manages internally — `_data`, `_idx`, `_config`, and a few others depending on the schema. Wrangler's export path sees these and refuses the whole database.

Workaround: don't use `wrangler d1 export`. Enumerate your user tables from `sqlite_master`:

```sql
SELECT name FROM sqlite_master WHERE type = 'table' ORDER BY name;
```

This lists shadow tables too, so filter out the FTS virtual table and the shadows it created (they'll all share its name as a prefix). The regular user tables are the ones you want.

Then `wrangler d1 execute <db-name> --command "SELECT * FROM <table_name>"` per table for the data. More calls, but it works.

The FTS virtual table doesn't need to be in the backup — it's rebuilable from the source data. The original `CREATE VIRTUAL TABLE` DDL is all you need to recreate it on restore.

Annoying that `wrangler d1 export` fails on the whole database rather than skipping virtual tables gracefully. At least the error message names the reason.
