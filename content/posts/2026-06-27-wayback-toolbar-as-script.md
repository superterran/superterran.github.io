---
title: "wayback toolbar files loaded as scripts"
date: 2026-06-27
draft: false
categories: ["homelab"]
tags: ["wayback", "museum", "javascript", "gotcha", "playwright"]
summary: "Archiving old site snapshots from Wayback? The .js files it saved might be HTML toolbar pages — browsers throw 'Unexpected token <' and the error points at nothing useful."
---

Running a Playwright QA pass over the museum archive. Loaded a 2008-era snapshot page. Five JS exceptions in the console, all the same shape:

```
Uncaught SyntaxError: Unexpected token '<'
```

All from `superterran-2008/assets/*.js`.

First assumption: bad JS from 2008. The reality: those aren't JS files. When Wayback Machine crawled the original site, some referenced scripts were already dead. Wayback saved those failed fetches as HTML — the Wayback toolbar wrapper page — with the original `.js` filename intact. So the assets directory has files with `.js` extensions that contain HTML.

A browser loading them as `<script src>` gets HTML, hits the `<` in `<!DOCTYPE`, and throws. The error points at the filename. It doesn't say "this file contains HTML." So first instinct is to read the script logic — there is no script logic to read.

Fix was deleting the stub files. The scripts they referenced were already dead at crawl time; removing them cleared all five exceptions.

Same thing can happen with `.css` and image files. Wayback saves the toolbar HTML whatever the original extension was. If a crawled resource was a 404 at crawl time, the saved asset is an HTML page wearing the wrong filename.
