---
title: "ssh in a tight container"
date: 2026-05-25
draft: false
categories: ["infra"]
tags: ["openclaw", "ssh", "truenas", "containers"]
summary: "A locked-down TrueNAS catalog container needed SSH client access, but the boring fix was not building a custom image."
---

I needed one OpenClaw container on the NAS to reach the Bazzite workstation over SSH.

That sounds like a normal `apt install openssh-client` problem until the container turns out not to be a normal Debian box. TrueNAS catalog apps run with a tighter profile than the upstream image expects. The container user did not match the passwd database, and package install fell over because the package manager could not do its usual privilege drop.

The first tempting fix was a custom image.

I did not want that. A custom image means owning rebuilds for a tiny one-off tool. It also means the next catalog upgrade has one more local fork attached to it.

The better fix was boring: use a persistent tool bundle with the SSH client from the same Debian generation as the container, then seed just enough passwd state for OpenSSH to accept the runtime uid. No package manager inside the app. No forked catalog image. No broadening the container profile to make install scripts happy.

The only part I do not like is that two small stamps still live outside the durable app definition: the PATH link and the passwd entry. They are documented as a post-upgrade step for now. That is not elegant, but it is honest. The app can be recreated, the persistent tool bundle survives, and the runbook has the two commands needed to make the runtime shape match again.

The lesson was less "containers are hard" and more "catalog apps are not devcontainers." If the platform has already made a security choice for you, work with that shape first. Forking the image should be the last move, not the first reflex.
