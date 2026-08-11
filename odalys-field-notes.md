<!--
  My onboarding journey log — tracked in git so it reaches the maintainer.
  My AI assistant appends dated entries here automatically as I hit friction
  (see CLAUDE.local.md). I can also write in it myself anytime.
-->

# odalys — Bloob Haus onboarding field notes

This is my running log of building my own Bloob Haus site: what I had to change, what confused me, what
didn't work, and what I wished "just worked." It's raw and append-only — **not** a polished doc. The
maintainer reads it to make onboarding smoother for the next person.

Newest entries at the top.

---

## 2026-07-29 — first local render: three first-run snags on Linux
Ran `dev-local.js --site=odalys` against my real vault (29 notes). It built and served at
`localhost:8080` — individual note pages render great. Three things a first-timer hits:

1. **Theme watcher crashes on Linux / Node 18.** `scripts/watch-themes.js` uses recursive `fs.watch`,
   which throws `ERR_FEATURE_UNAVAILABLE_ON_PLATFORM` here (recursive watch on Linux needs Node 20+).
   Eleventy's own server still runs and hot-reloads *content* edits, but *theme-file* edits won't
   auto-rebuild until I restart the dev server. Almost certainly invisible on macOS (where Leon dev's,
   recursive watch works there). Shared-infra bug → candidate for an upstream PR (chokidar or a
   graceful non-recursive fallback).

2. **A fresh vault has no homepage — `/` returns 404** ("not found"). With no root `index.md`, the
   site has no landing page even though all 29 note pages work. Obsidian's default new-note is
   `Welcome.md`, which lands at `/Welcome/`, not `/`. A first-timer reasonably expects *something* at
   the root. Fix is easy (add `index.md`) but it isn't obvious, and nothing in the build warned me.

3. **Publish-mode logging is misleading when unconfigured.** With no `_bloob-settings.md` and no
   `publish_mode` in the site yaml, the filter logged `Mode: undefined` then `Allowlist: publish: true`
   — but actually behaved as blocklist-publish-everything (all 29 notes published, including private
   ones like `car insurance`). Fine for a localhost preview, but the log contradicts the behavior, and
   for a safety-critical filter that's exactly where you don't want ambiguity. Reinforced that I need
   an explicit publish mode set before any deploy.

4. **Stopping the dev server needs to kill the whole subtree, not just `dev-local.js`.** The CLAUDE.md
   "Dev Server Rule" says to stop it with `pkill -f "dev-local.js"`, but `dev-local.js` spawns
   `concurrently` → `eleventy --serve` via `execSync`, and killing only the parent orphans that
   subtree. Result: the old eleventy kept holding port 8080, so a restart silently bumped to 8081 and
   I briefly had two servers running. Reliable stop needs the eleventy/concurrently children too (or a
   process-group kill). Worth updating the documented stop command. **Working stop that avoids
   orphans:** `kill -TERM -$(ps -o pgid= -p $(lsof -ti :8080) | tr -d ' ')` — kills the whole process
   group of the port-8080 listener in one shot.

5. **A fresh site has no automatic homepage navigation.** After publishing several notes, nothing on
   the homepage links to them — `melt`'s `nav.njk` is a breadcrumb bar (only shows on non-home pages),
   and `home.njk` is built around a `bubble-nav` visualizer that isn't built yet (its comment says it
   falls back to a link list). So the homepage nav has to be hand-authored as a markdown list in the
   vault's `index.md`. It works well (wiki-links resolve cleanly, 0 broken), but it's manual — a
   first-timer expects *some* default index/nav of published pages, and has to discover that they must
   build it themselves. The auto-generated folder-index pages (`/quotes/`, etc.) DO auto-list their
   contents via `folder-preview`, so a lighter default homepage could reuse that.

## 2026-07-29 — first session: clone → install → local dev
Cloned the pre-staged `odalysbest/bloob-haus-webapp` fork. `sites/odalys.yaml`, the deploy workflow, and
`collaborating.md` were already filled in by Leon, which made the start smooth. `npm install` ran clean on
Node 18.19.1 (some npm audit warnings, nothing blocking). Next: pointing local dev at my own vault folder
to see the site render for the first time.
