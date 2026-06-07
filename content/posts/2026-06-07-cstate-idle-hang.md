---
title: "when the log goes quiet at 0% cpu"
date: 2026-06-07
draft: false
categories: ["homelab"]
tags: ["bazzite", "amd", "zen5", "kernel", "troubleshooting"]
summary: "Three debug sessions blamed the wrong thing. The real tell was an 8-hour journal gap with 53°C CPU and zero kernel output."
---

Bazzite desktop hard-locked again overnight. No panic, no MCE, no thermal alarm, no suspend entry — just a gap in the journal and a box that needed a hard reset. Third session in a week chasing this thing.

The first fix was a fan curve (June 3). The second was pulling some forced-EDID kargs that were causing KMS hangs (June 5). The third was a staged NVIDIA driver upgrade for a GSP heartbeat timeout. All real problems. None of them was the freeze.

The tell was in the timestamps. `journalctl --list-boots` showed boot -1's last entry at 00:08:13. The hard reset happened at 08:19:50. Eight hours of silence.

Netdata filled in the rest. CPU was at ~0% when the collector flatlined around 00:02–00:04. k10temp Tctl was 53°C — well below any thermal threshold. No load. No heat. Just stopped.

That's the fingerprint: **cold, idle, log-less, no panic, needs a hard reset**. It's not a crash. It's a C-state hang. The AMD Zen idle deep-sleep path (C3 on a 9950X, BIOS from Jul-2025 on a B850-I) can wedge the CPU hard enough that nothing recovers it — no watchdog fires, no pstore entry, no Xid, nothing. The box is gone.

Three things I wired up as a response:

```bash
# 1. hardware watchdog — auto-reboots in ~1 min instead of 8hr dead
# /etc/systemd/system.conf.d/10-watchdog.conf
RuntimeWatchdogSec=60s

# 2. soft lockup → panic → reboot → log
# /etc/sysctl.d/99-hang-capture.conf
kernel.softlockup_panic=1
panic_on_oops=1
panic=10

# 3. disable deep C-states live (bridge until karg takes effect)
for cpu in /sys/devices/system/cpu/cpu*/cpuidle/state{2,3}; do echo 1 > "$cpu/disable"; done
```

And queued `processor.max_cstate=1` as an rpm-ostree karg for the next reboot.

The real fix is a BIOS update (newer AGESA) plus setting Power Supply Idle Control to "Typical Current Idle" in UEFI. The karg is a bridge until that's confirmed stable — after that, deep C-states can be re-enabled.

One gotcha: `systemctl show -p RuntimeWatchdogSec --value` returns blank even when the watchdog is armed. The actual property is `RuntimeWatchdogUSec`. A blank from the first command doesn't mean the watchdog isn't running.

Netdata is worth mentioning here. If it's running and retains a few days of data, you can query `system.cpu` and the k10temp sensor around the journal gap and reconstruct exactly what the machine was doing at freeze time. That's what ruled out thermal and load definitively — not inference, just a flat line on both graphs at the moment the collector stopped.
