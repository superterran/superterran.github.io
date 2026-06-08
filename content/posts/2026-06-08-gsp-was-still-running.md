---
title: "the c-state karg didn't hold"
date: 2026-06-08
draft: false
categories: ["homelab"]
tags: ["bazzite", "nvidia", "gsp", "kernel", "troubleshooting"]
summary: "The max_cstate=1 mitigation was active when the box froze again. The watchdog also didn't fire. And the June 5 'GSP fix' never actually disabled GSP."
---

Froze again yesterday. Two hours of uptime, light load, `processor.max_cstate=1` active in the kernel cmdline.

That's a problem for the idle C-state theory. The mitigation was supposed to keep the CPU out of deep idle states, but the freeze happened anyway — under actual use, not overnight idle.

Also: the watchdog didn't fire.

[Yesterday's post](/posts/2026-06-07-cstate-idle-hang/) mentioned wiring up the SP5100 TCO watchdog at a 60s timeout, confirmed armed. The boot gap this time was about two minutes — last journal entry at 12:32:09, first entry after reset at 12:34:03. That's inside the watchdog window. But the recovery was a manual power-down. journald went silent at 12:32 with no timer or journald entries after that, which means systemd itself wedged — and the TCO reset path wedged with it. A hang this deep doesn't leave a pstore entry, doesn't trip soft-lockup detection, doesn't let the hardware watchdog do anything useful.

There's also a correction on the NVIDIA side. A session on June 5 supposedly addressed the GSP issue by upgrading the driver to 610.43.02. It didn't, really. The upgrade swapped the driver binary but left GSP running — `GPU Firmware: 610.43.02` was still loading in the logs. Disabling GSP requires a kernel argument:

```
nvidia.NVreg_EnableGpuFirmware=0
```

A newer driver version and a GSP disable are different things. The June 5 session conflated them, so GSP has been running the whole time.

That karg is staged now alongside the others:

```
kargs --pending
→ processor.max_cstate=1, split_lock_detect=off, nvidia.NVreg_EnableGpuFirmware=0
```

The freezes are happening ~1–2h after boot under light active load, which fits a GSP firmware heartbeat timeout better than idle C-state wedging. If it freezes again with GSP confirmed off, that theory is also wrong. Next lever after that is Power Supply Idle Control in UEFI — something separate from the OS-side karg that hasn't been touched yet.

Reboot pending. Waiting to see if it holds.
