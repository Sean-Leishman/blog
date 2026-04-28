# Content workflow

How posts actually get written and shipped on this blog.

## Sections
Two top-level content sections, both listed on the homepage:
- `content/blogs/` — long-form posts.
- `content/projects/` — project retrospectives, usually with a link to a GitHub repo and a hero image.

`config.yml` pins these via `params.mainSections: ["blogs", "projects"]`.

## Frontmatter
The `hugo new` archetype is minimal:
```yaml
---
title: "..."
date: <auto>
draft: true
---
```
Existing posts use a richer pattern. Copy this when starting a new one:
```yaml
---
author: "Sean Leishman"
title: "Post title"
date: "YYYY-MM-DD"
tags: ["Tag1", "Tag2"]
Summary: "One-line summary used on listings and OG cards."
ShowBreadCrumbs: true
ShowToc: false
hideMeta: false
# weight: 1          # only for projects, controls listing order
# math: true         # only if the post uses KaTeX
---
```

Notes:
- `author` is set per post; there's no global default.
- `tags` are title-case strings.
- Project posts use `weight` to control ordering on the projects listing.
- Add `math: true` to opt into KaTeX rendering for that post (site-level `math: true` is also on, so this is mostly belt-and-braces).

## Images
1. Drop the file into `static/` (e.g. `static/myproject.png`).
2. Reference it from a post:
   ```
   {{< figure align=center width=80% height=auto src="../../myproject.png" >}}
   ```
   `figure` is PaperMod's built-in shortcode. The `../../` path resolves correctly because `static/` becomes the site root.
3. For social/OG previews, set `images: ["myproject.png"]` in frontmatter, or rely on the site-level `params.images: ["papermod-cover.png"]`.

## Linking out
Use the local shortcode for external links so they open in a new tab safely:
```
{{< newtabref href="https://example.com" title="Example" >}}
```
Defined in `layouts/shortcodes/newtabref.html`. Renders `<a target="_blank" rel="noopener">`.

## Math
Inline: `\( ... \)` or `$ ... $` (depends on KaTeX auto-render config — site uses defaults).
Display: `\[ ... \]` or `$$ ... $$`.

KaTeX is loaded from `layouts/partials/math.html`. See `docs/notes.md` re: the two KaTeX versions currently loaded.

## Drafting
- `draft: true` keeps a post out of builds (`buildDrafts: false`).
- Preview drafts: `hugo server -D`.
- Future-dated posts are also hidden until the date passes (`buildFuture: false`).
- Expired posts (`expiryDate` past) are excluded too (`buildExpired: false`).

## Publishing
1. Set `draft: false` (or remove the key).
2. Commit + push to `master`.
3. CI builds and deploys to https://blog.seanleishman.com within a couple of minutes.

## Archive page
`content/archives.md` mounts the PaperMod `archives` layout at `/archives/`. New posts show up automatically — no edits needed there.

## Taxonomies
Configured: `categories`, `tags`, `series`. Only `tags` is used in practice. `series` would group multi-part posts if any existed.
