---
title: "sunshine flatpak won't do kms capture"
date: 2026-06-10
draft: false
categories: ["homelab"]
tags: ["sunshine", "moonlight", "bazzite", "kms", "flatpak"]
summary: "Moonlight showed the app list but every stream failed. The flatpak build can't hold CAP_SYS_ADMIN, so KMS capture is structurally off the table."
---

Moonlight showed the app list. Every stream attempt died immediately with "failed to initialize display." Sunshine was running — `systemctl --user status` showed active, no crash. Logs said nothing obvious at first glance.

The day before, as part of a headless streaming experiment, the brew-native Sunshine had been uninstalled and replaced with the Flatpak. That was the problem.

The log, once I looked in the right place (`~/.var/app/dev.lizardbyte.app.Sunshine/.../sunshine.log`):

```
Fatal: AppImage and Flatpak do not support KMS capture
WAYLAND_DISPLAY has not been defined
```

The Flatpak runs inside a `bwrap` sandbox. KMS capture requires `CAP_SYS_ADMIN` — you need direct access to the DRI device. The bwrap sandbox structurally can't hold that capability. So the Flatpak build will start, list apps, accept connections, and then fail silently on capture every time. No stream, no error surfaced to the client.

`ujust setup-sunshine` on Bazzite shows the supported path: the brew package via `LizardByte/homebrew`, not the Flatpak. The postinst script sets the caps on the binary directly:

```
cap_sys_admin,cap_sys_nice=p
```

No sandbox. Direct device access. That's the only path that works for KMS mirroring on a bare-metal Bazzite box.

Uninstalled the Flatpak, pulled the brew package, ran the postinst, wrote a clean config with `capture = kms` and `encoder = nvenc`. First start showed "Screencasting with KMS" in the log. Streams worked.

Worth noting: a fresh install drops the previous client certs. The web UI needs a new admin credential set before Moonlight will pair.
