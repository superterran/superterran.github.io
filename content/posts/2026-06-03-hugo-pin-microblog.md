---
title: "why micro.blog theme previews need old hugo"
date: 2026-06-03
draft: false
categories: ["hugo"]
tags: ["micro.blog", "hugo", "theme", "tooling"]
summary: "Building a micro.blog theme locally with modern Hugo breaks immediately — micro.blog pins Hugo at 0.91 and the theme uses idioms removed in later versions."
---

System Hugo is 0.147. Run `hugo server` against a micro.blog theme and it dies:

```
ERROR .Site.Author was removed in Hugo v0.124.0
```

Or if you've wired up `contentDir` correctly:

```
ERROR paginate was removed in Hugo v0.128.0. Use [pagination] instead.
```

Neither of these is a bug in the theme. micro.blog runs Hugo somewhere around 0.91 internally, and the themes it hosts were written for that. `.Site.Author` is a template variable Hugo removed in 0.124. `paginate` in `config.json` became `[pagination] pagerSize` in 0.128.

The wrong move is patching the theme to silence the errors. If you rename `.Site.Author` to `.Site.Params.author`, the change is now in your working tree. It may even render locally. But the deployed build runs on micro.blog's Hugo — which doesn't have the new variable. The patch doesn't fix the theme; it breaks the deployed build and leaves you with a local environment that diverges from production.

The right move is pinning a Hugo 0.91.2 binary for local preview work:

```sh
curl -L \
  https://github.com/gohugoio/hugo/releases/download/v0.91.2/hugo_extended_0.91.2_Linux-64bit.tar.gz \
  | tar xz -C bin/ hugo
mv bin/hugo bin/.hugo
echo "bin/.hugo" >> .gitignore
```

Use `./bin/.hugo` instead of the system `hugo` for anything theme-related. The errors disappear. The rendered output matches what micro.blog produces.

The theme.toml says `min_version = "0.91"`. Don't build against anything newer until micro.blog does.
