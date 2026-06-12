---
title: "building a local voice ai stack with open webui, kokoro, and speaches"
date: 2026-04-11
draft: false
categories: ["homelab"]
tags: ["ollama", "open-webui", "kokoro", "speaches", "cloudflare", "bazzite", "voice-ai"]
summary: "Open WebUI for the interface, Kokoro for TTS, Speaches (faster-whisper) for STT, Ollama for the model. One VRAM constraint, one auth pattern, one Cloudflare gotcha."
---

The local voice AI stack on bazzite-desktop: Open WebUI as the interface, Kokoro for text-to-speech, Speaches (faster-whisper) for speech-to-text, all backed by a locally-running large model via Ollama. Exposed publicly through the existing Cloudflare Tunnel with two different auth patterns for the web UI vs. API endpoints.

## VRAM budget forces Kokoro onto CPU

The machine is running a large model in Ollama that holds most of the GPU's VRAM. Kokoro's GPU image needs several gigabytes that aren't available while Ollama is running.

The good news: Kokoro TTS is a small model (82M parameters). It runs fine on CPU — slower than GPU, but the latency for speech synthesis is acceptable for conversational use. The CPU variant of the Kokoro Docker image solves the contention without requiring any scheduling coordination.

Speaches (faster-whisper STT) gets GPU access since its VRAM footprint is lighter. The `medium` whisper model fits in the remaining headroom.

## Two auth patterns: web UIs vs. API endpoints

Web UIs (Open WebUI itself) go through Cloudflare Access with email OTP. The result: visiting the URL prompts for a one-time code sent to the authorized email. No credentials embedded anywhere, no password database to maintain.

API endpoints — specifically the Ollama API, which external clients like iOS apps and scripts need to hit programmatically — can't do OTP flows. Those go through Caddy as a bearer token proxy: Caddy sits in front of Ollama, validates an `Authorization: Bearer ...` header, and proxies to the local Ollama socket. The public tunnel routes API traffic through Caddy.

## Remotely-managed tunnel configuration

This was the unexpected part. The Cloudflare Tunnel is "remotely managed" — ingress rules are stored in Cloudflare's API, not in a local `config.yml`. Editing the local config file does nothing; the remote config is what actually controls routing.

Adding new subdomains requires an API call to update the remote ingress config. Worth checking whether your tunnel is `local` or `cloudflare`-managed before spending time editing a file that isn't being read.

## The catch-all redirect rule

The zone had a dynamic redirect rule catching all traffic to unrecognized `*.superterran.net` subdomains and 301-redirecting to another domain. New subdomains added to the tunnel ingress weren't exempt from this rule — requests to the new hostnames hit the redirect before reaching the tunnel.

Fix: add each new hostname to the catch-all rule's exclusion list before wiring it into the tunnel. The symptom (redirect to unexpected domain instead of the expected service) is confusing if you haven't seen this pattern before.

## Open WebUI audio configuration

Open WebUI's STT and TTS integrations are in Admin → Settings → Audio. The fields expect OpenAI-compatible endpoints — Speaches and Kokoro both implement the OpenAI audio API surface (`/v1/audio/transcriptions`, `/v1/audio/speech`), so they drop in directly. Set the base URLs and the model names, and voice mode works.
