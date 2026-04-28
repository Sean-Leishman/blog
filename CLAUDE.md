# blog

## Purpose
Source for Sean Leishman's personal blog at https://blog.seanleishman.com — a static site of long-form writeups (under `content/blogs`) and project retrospectives (under `content/projects`). Built with Hugo and the PaperMod theme, deployed to Firebase Hosting on the `portolfio-2c85b` project (site `sean-blog`).

## Tech stack
- Hugo (static site generator) — config in `config.yml`, no `hugo.toml`.
- PaperMod theme, pulled in as a git submodule at `themes/PaperMod`.
- Firebase Hosting (`firebase.json`, `.firebaserc`) for deploy.
- GitHub Actions for CI: Hugo build + Firebase deploy.
- KaTeX (loaded from CDN via `layouts/partials/math.html`) for math rendering when `math: true` is set.
- jQuery + custom signature SVG animation (`layouts/partials/extend_footer.html`).

## Key files / entry points
- `config.yml` — site config: `baseURL`, theme, taxonomies, menu, params (homeInfo, social, fonts, math).
- `content/blogs/` — published blog posts (`welcome.md` is the only one so far).
- `content/projects/` — project writeups (`chess.md`, `et-tu-uv.md`, `gryosound.md`, `stocktrend.md`).
- `content/archives.md` — uses the `archives` layout from PaperMod.
- `archetypes/default.md` — frontmatter template for `hugo new`.
- `layouts/partials/extend_head.html` — adds jQuery, IBM Plex Mono / Nanum Gothic Coding fonts, conditionally KaTeX.
- `layouts/partials/extend_footer.html` — SVG signature draw animation.
- `layouts/partials/math.html` — KaTeX loaders.
- `layouts/partials/post_meta.html` — overrides PaperMod's post meta line.
- `layouts/shortcodes/newtabref.html` — `{{< newtabref href="…" title="…" >}}` shortcode for external links.
- `assets/css/extended/custom.css` — site-specific style overrides on top of PaperMod.
- `static/` — favicons, post images (`chess.png`, `gyroArch.png`, etc.), avatar `ReadyPlayerMe-Avatar(2) - Copy.png`.
- `firebase.json` — hosting config; serves `public/` to site `sean-blog`.
- `.github/workflows/hugo.yml` — primary deploy (push to `master`): builds with Hugo and deploys via FirebaseExtended action.
- `.github/workflows/firebase-hosting-merge.yml` / `firebase-hosting-pull-request.yml` — Firebase-CLI-generated workflows (also run `hugo`, then deploy).

## How to run / dev
- `hugo server` — local dev server with live reload.
- `hugo` — build into `public/` (gitignored).
- First clone: `git submodule update --init --recursive` to pull PaperMod.
- New post: `hugo new blogs/<slug>.md` or `hugo new projects/<slug>.md`. Drafts have `draft: true`; flip to `false` (or just remove) to publish.
- Manual deploy (rare; CI normally handles it): `hugo && firebase deploy`.

## Conventions
- Two content sections only: `blogs` and `projects` (set in `mainSections` in `config.yml`).
- Frontmatter uses YAML with capitalised PaperMod keys: `Summary`, `ShowBreadCrumbs`, `ShowToc`, `hideMeta`. Author is hardcoded `"Sean Leishman"` per file.
- Tags are title-case (`["Project", "Chess", "Python", "Pygame", "ML"]`).
- Project posts link to source via the `newtabref` shortcode and embed a hero image with PaperMod's `figure` shortcode pointing at `static/`.
- Math posts opt in by adding `math: true` to frontmatter; site-level `math: true` is also on so it's effectively always loaded.
- Theme is consumed as a submodule — don't edit files under `themes/PaperMod`; override via `layouts/` and `assets/css/extended/`.

## Gaps / honesty
- Three GitHub Actions workflows exist and partially overlap; `firebase-hosting-pull-request.yml` even has a malformed step (two `uses:` keys). The active deploy path appears to be `hugo.yml`.
- Hugo version pinning is inconsistent: `hugo.yml` declares `HUGO_VERSION: 0.114.0` as env but installs `0.64.0` via `peaceiris/actions-hugo`.
- `package-lock.json` is empty / vestigial — there are no Node deps.
- Working tree currently shows `themes/PaperMod` as having modified content (submodule drift); cause not investigated here.
- No tests, linters, or formatters configured.
