---
title: "publishing a devcontainer template to ghcr with oci distribution"
date: 2026-05-05
draft: false
categories: ["homelab"]
tags: ["devcontainers", "docker", "ghcr", "oci", "github-actions", "mkcert"]
summary: "Option D architecture for devcontainer templates: working repos publish directly to a shared GHCR namespace. Three gotchas: cross-repo package ownership, Docker Compose variable syntax conflict, and a broken devcontainers-cli feature."
---

The devcontainers spec supports publishing templates and features to any OCI registry. The publish toolchain is `devcontainer templates publish` — but getting it wired up involves a few non-obvious decisions about registry layout and build tooling.

## Option D: working repos publish to a shared namespace

There are a few viable architectures for hosting devcontainer templates. The one that worked here:

- A shared GHCR namespace (`ghcr.io/<username>/devcontainer-templates`) holds all published template OCI artifacts
- Each project repo contains its own template source under `.devcontainer/`
- A GitHub Actions workflow in each project repo runs `devcontainer templates publish` on push, targeting the shared namespace

This keeps template source co-located with the project that uses it, while making the published artifact discoverable from a single namespace. No separate "templates monorepo" required.

## GHCR cross-repo package ownership

Publishing to `ghcr.io/<username>/devcontainer-templates` from a repo other than `<username>/devcontainer-templates` requires an explicit package ownership grant. GHCR ties package write access to repository identity by default — a workflow in `my-project` cannot write to the `devcontainer-templates` package without being listed as an authorized repository.

This grant lives in the GHCR UI under the package settings, not in GitHub Actions or repository settings. The workflow token needs `packages: write` permission and the source repo needs to be added to the package's linked repositories list. Without the UI grant, the push fails with a 403 even if the token has the right scopes.

## Docker Compose variable syntax conflict

The devcontainers `devcontainer.json` template spec uses `${templateOption:optionName}` syntax for template variables. Docker Compose uses `${VARIABLE_NAME}` syntax for environment variable substitution. When a `devcontainer.json` references Docker Compose files and also contains `${templateOption:...}` tokens, the Compose parser intercepts and rejects the template variable syntax before the devcontainer toolchain can process it.

The fix: don't rely on the devcontainer CLI to do template variable substitution at build time when Compose is involved. Instead, a `build.sh` script applies the substitutions manually (simple `sed` replacements) on the template source before passing it to `devcontainer templates publish`. The published artifact contains the already-substituted values. This means one published variant per option combination rather than a single parameterized artifact, but for a small option matrix that's fine.

## Broken devcontainers-cli feature

The `devcontainers-extra/features/devcontainers-cli:1` feature installs the `@devcontainers/cli` npm package inside the container. At time of writing, the feature script references the `lts` npm dist-tag, which doesn't exist for this package — `lts` is a Node.js concept, not an npm package dist-tag. The install fails with a 404.

Fix: install `@devcontainers/cli` directly via a `postCreateCommand` or a custom feature, skipping the broken community feature entirely. `npm install -g @devcontainers/cli@latest` works fine.

## Multi-arch mkcert

mkcert is distributed as pre-built binaries. The install script needs to detect the architecture and pull the correct binary. The standard pattern:

```bash
ARCH=$(uname -m)
case $ARCH in
  x86_64)  MKCERT_ARCH="amd64" ;;
  aarch64) MKCERT_ARCH="arm64" ;;
  *)       echo "unsupported arch: $ARCH"; exit 1 ;;
esac
curl -Lo /usr/local/bin/mkcert \
  "https://github.com/FiloSottile/mkcert/releases/latest/download/mkcert-v$(...)_linux-${MKCERT_ARCH}"
```

Skipping the arch detection works fine until someone builds on an Apple Silicon machine or a Graviton runner. Worth doing upfront.
