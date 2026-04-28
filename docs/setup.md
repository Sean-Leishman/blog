# Setup

Personal notes for getting the blog running locally and deploying.

## Prerequisites
- Hugo. CI uses `peaceiris/actions-hugo@v2` pinned to `0.64.0`; locally any reasonably recent Hugo works (extended not required — site uses CDN KaTeX, not Sass-bundled icons beyond PaperMod's own).
- Git (with submodule support).
- Firebase CLI (`npm i -g firebase-tools`) — only if deploying manually. CI normally handles it.

## First-time clone
```
git clone <repo>
cd blog
git submodule update --init --recursive   # pulls themes/PaperMod
```

If `themes/PaperMod` is empty, Hugo will error with "module not found" — re-run the submodule init.

## Local dev
```
hugo server
```
Defaults to http://localhost:1313. Live reload included. `buildDrafts: false` in `config.yml`, so posts with `draft: true` won't show — pass `-D` to preview them: `hugo server -D`.

## Build
```
hugo
```
Output goes to `public/`. The directory is gitignored.

## Authoring

New post:
```
hugo new blogs/<slug>.md
hugo new projects/<slug>.md
```
Uses `archetypes/default.md`, which produces:
```yaml
---
title: "..."
date: <auto>
draft: true
---
```
Existing posts use a richer frontmatter (see `CLAUDE.md` / `docs/content-workflow.md`). Copy from a sibling post rather than relying on the archetype.

Images: drop into `static/`, then reference relatively from a post (e.g. `src="../../chess.png"`). Hugo copies `static/` to the site root.

## Deploy

### Automatic (preferred)
Push to `master`. Two workflows fire:
- `hugo.yml` — full Hugo build, deploys via `FirebaseExtended/action-hosting-deploy@v0`.
- `firebase-hosting-merge.yml` — Firebase CLI's own workflow, also deploys to channel `live`.

Both target Firebase project `portolfio-2c85b`, site `sean-blog`. Effectively redundant — see `docs/notes.md`.

### Manual
```
hugo
firebase deploy --only hosting
```
Requires being authenticated (`firebase login`) and having access to project `portolfio-2c85b`.

## Domain
Live at https://blog.seanleishman.com. The `baseURL` in `config.yml` is `https://www.blog.seanleishman.com`; the `www` variant matters for sitemap / canonical URLs. Domain mapping is configured in the Firebase console (not in this repo).

## Updating PaperMod
```
cd themes/PaperMod
git fetch origin
git checkout <tag>
cd ../..
git add themes/PaperMod
git commit -m "Bump PaperMod"
```
Don't edit files inside `themes/PaperMod` — overrides go in `layouts/` and `assets/css/extended/`.
