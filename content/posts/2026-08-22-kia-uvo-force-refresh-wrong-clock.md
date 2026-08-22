---
title: "kia_uvo force_refresh_interval measures the wrong clock"
date: 2026-08-22
draft: false
categories: ["homelab"]
tags: ["home-assistant", "kia_uvo", "automation", "hyundai"]
summary: "The built-in force-refresh gate compares against the car's last report timestamp, not the wall clock — so once the car goes quiet, every soft poll silently becomes a hard API call."
---

Set up a Home Assistant automation to watch my 2025 Tucson while it was at the shop. The `kia_uvo` integration has a `force_refresh_interval` option — fire a hard API call to wake the car every N minutes instead of just reading the cloud cache. The intent is five hard checks a day, not sixty.

It doesn't work that way when the car stops reporting.

The gate logic is roughly:

```
if (now - vehicle.last_updated_at) > force_refresh_interval:
    force_refresh()
```

`vehicle.last_updated_at` is the car's last report timestamp, not when HA last talked to the cloud. When the car is active the distinction doesn't matter. When it goes quiet — sleeping in a shop bay, 12V potentially disconnected — `last_updated_at` freezes. The condition `(now - frozen_ts) > force_refresh_interval` becomes permanently true after the first interval elapses. So every 10-minute soft poll triggers a force refresh. 60+ hard pings a day.

The fix: set `force_refresh` to something absurd (I used 525600, which is one year in minutes) to disable the built-in path, then drive hard checks from an explicit automation on a fixed schedule. Five calls to `button.{car}_force_refresh` at named times. Five per day regardless of what the car's clock says.

One wrinkle: `kia_uvo` options are UI-only in current HA — there's no YAML path. To set them without the frontend: stop HA, patch `.storage/core.config_entries` directly, restart. HA rewrites `.storage` on shutdown, so a bare restart won't take the edit. You need a clean stop first.
