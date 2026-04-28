# Notes

Loose observations and known sharp edges. Not a polished doc.

## Three deploy workflows
There are three workflows in `.github/workflows/`. At least two of them deploy to Firebase on every push to `master`:
- `hugo.yml` — explicit, pinned Hugo, uploads a Pages artifact (vestigial — the Pages step doesn't actually publish anywhere) and deploys to Firebase.
- `firebase-hosting-merge.yml` — Firebase-CLI-generated, runs `hugo` then `FirebaseExtended/action-hosting-deploy@v0` to channel `live`.

Picking one and deleting the other would be tidier. `hugo.yml` is the more deliberate of the two.

`firebase-hosting-pull-request.yml` has a bug: the final step contains two `uses:` keys and a stray `run: hugo`. It likely silently fails on PRs.

## Hugo version mismatch
`hugo.yml` sets `env.HUGO_VERSION: 0.114.0` but the `peaceiris/actions-hugo` step pins `hugo-version: "0.64.0"`. The env var is unused. Effective build is on Hugo 0.64.0, which is ancient. If anything PaperMod-version-sensitive starts failing, suspect this.

## Two KaTeX versions loaded
`layouts/partials/math.html` loads:
- katex 0.11.1 JS + auto-render.min.js (deferred)
- katex 0.16.4 CSS, JS, and auto-render.min.js (also deferred)

Both auto-render scripts call `renderMathInElement(document.body)` on load. Probably benign (last to run wins) but it's loading two copies of KaTeX's JS for no reason. Drop the 0.11.1 block.

## `package-lock.json` is empty
```json
{ "name": "blog", "lockfileVersion": 3, "requires": true, "packages": {} }
```
No `package.json`, no Node deps. Safe to delete.

## Firebase project ID is misspelt
`.firebaserc` and the workflows reference `portolfio-2c85b` (note: "portolfio", not "portfolio"). It's the actual project ID in Firebase, so don't "fix" it locally — you'd just break deploy.

## Submodule drift in working tree
`git status` shows `modified: themes/PaperMod (modified content)`. The submodule's working tree has uncommitted changes. Could be local theme tweaks that should have been moved to `layouts/` / `assets/css/extended/`, or just an artifact of an aborted update. Worth a `cd themes/PaperMod && git status` before doing anything submodule-related.

## Mixed-case frontmatter keys
PaperMod accepts both lowercase (`title`, `date`) and TitleCase (`Summary`, `ShowToc`, `ShowBreadCrumbs`) keys. Existing posts mix them. Hugo doesn't care; just be consistent within a post.

## Avatar filename has a space and parens
`static/ReadyPlayerMe-Avatar(2) - Copy.png` — referenced from `config.yml` as `params.label.icon`. Renaming would require updating `config.yml`.

## `mainSections`
Set to `["blogs", "projects"]`. If a third top-level content section is added, remember to add it here or the homepage list won't pick it up.

## Custom signature animation
`layouts/partials/extend_footer.html` runs on every page and unconditionally calls `document.querySelector("#signature")`. If no `#signature` element is on the page, `observer.observe(target)` is called with `null` and throws in the console. Currently only fires on pages that include the SVG.

## RSS / JSON outputs
`outputs.home` includes RSS and JSON. There's no link in the nav to either; consumers would need to know `/index.xml` or `/index.json`.

## Drafts behaviour
`buildDrafts: false`, `buildFuture: false`, `buildExpired: false` — strict. Posts dated in the future will be invisible until the date passes. Useful but easy to forget if you backdate.
