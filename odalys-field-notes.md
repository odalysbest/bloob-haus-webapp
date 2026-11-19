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

## 2026-11-19 — attachments fix (PR), and one typo in a note kills the whole build
Built `features.referenced_attachments_only` (default off) and opened it as a PR. On my vault it
copies 14 of 239 attachments, exactly the images my published notes show, and 0 links break. It's
turned on in `sites/odalys.yaml`.

While testing, the build died with only `❌ Dev failed: can not read a block mapping entry; a multiline
key may not be an implicit key at line 5, column 1`. **No filename.** The cause was a missing space
in my homepage's frontmatter (`subtitle:work in progress.`). Finding it took a script that parsed every
note. Two suggestions: (1) the YAML error should name the note, and (2) for a *private* note, a
parse error probably shouldn't stop the build at all. Warning and skipping would be safer, and
skipping is also the safe direction in allowlist mode.

## 2026-11-18 — `npm run dev:odalys` dies with EMFILE (Linux inotify instance cap)
The dev server failed with only `❌ Dev failed: Command failed: npx concurrently ...`. The real
cause was buried hundreds of lines up: `EMFILE: too many open files, watch ...` from Eleventy's
chokidar watcher. It was **not** the open-file ulimit (1M) and not the watch count (360 / 65536). It was
Linux's `fs.inotify.max_user_instances`, which defaults to **128**, and my desktop apps (Obsidian, KDE,
Signal, Zoom...) already held 124. Fix: `sudo sysctl fs.inotify.max_user_instances=1024` (persisted in
`/etc/sysctl.d/`). Separately, the `[watch]` theme watcher still crashes on Node 18 (entry below,
08-10), which adds noise. **Suggestion:** `dev-local.js` could catch EMFILE/ENOSPC and print the sysctl
hint instead of the generic "Command failed". Linux-only, so it's probably invisible on macOS.

## 2026-11-18 — second upstream pull (170 commits), theme-fork drift, two publish-safety surprises
Merged `upstream/main` again (7 ahead / 170 behind). Zero merge conflicts, deploy workflow survived.
Afterward 2 tests failed, and both were exactly the drift the August fork note predicted: upstream moved
every theme's `head.njk` onto shared `seo-meta.njk` / `robots-meta.njk` partials, but `themes/odalys/`
is a `cp` of an old melt, so it was missed. I'd never customized those files, so I re-copied
`head.njk`, `footer.njk`, and `main.css` from melt and all 1631 tests passed. The tests did catch it,
which is good. **Suggestion:** a forked theme needs a "what changed in my parent theme since I forked"
step (record the fork commit in `theme.yaml`, plus a script that diffs against it).

Switched the vault to `publish_mode: allowlist` and rewrote `index.md` as the homepage nav. The filter
behaves (19 of ~30 notes published, 0 broken links). Two things surprised me:

1. **Stale build dirs become public folder pages.** `src-odalys/` only gets its `.md` files cleaned
   between builds, not its directories. Empty dirs from August's publish-everything build
   (`Collective-Livelihood-Garden/`, `Untitled/`, `Untitled-1/`) got auto-generated folder-index stubs,
   so the names of folders that are now private showed up as pages. Running `rm -rf src-odalys` fixed it,
   and CI is safe because it builds from a fresh clone. But it's a local-preview trap, and
   preprocess could prune empty dirs before Step 9.5.
2. **Attachments are published regardless of publish status.** `copyAttachments`
   (`scripts/utils/attachment-resolver.js`) globs *every* image/PDF in the vault into `_site`. They're
   not linked from anywhere, but they can be reached by URL, and Obsidian's `Pasted image <timestamp>.png`
   names are guessable. In allowlist mode, "private by default" doesn't hold for media. Example: a 7 MB
   screenshot that belongs to an unpublished note gets deployed. **Suggestion (shared infra, PR
   candidate):** in allowlist mode, copy only the attachments that published notes reference.

## 2026-08-10 — upstream pull (103 commits) + theme fork, both clean
Synced the fork with `upstream/main` — it was 2 ahead / 103 behind. The merge auto-resolved with
**zero conflicts** (223 files, +16.7k/-3.9k), and all 851 tests passed after `npm install`. Two things
that made it smooth and are worth keeping: (1) `sites/odalys.yaml` was byte-identical on both sides
(upstream had independently added it), so no conflict; (2) the `.gitattributes merge=ours` rule for
`.github/workflows/**` protected `deploy-odalys.yml` — but the driver was NOT configured on this clone,
so I had to run `git config merge.ours.driver true` first. **Suggestion:** the onboarding could mention
that the `ours` driver must be enabled per-clone *before the first merge*, not just eventually — it's
easy to merge once without it and not notice.

Then forked the theme: `cp -r themes/melt themes/odalys` + set `theme: "odalys"`. Minor friction —
the copied `theme.yaml` still said `name: melt` and "Built for melt.bloob.haus (Whitney & Vicki)", so
a fresh fork silently identifies as the client's theme until you edit it by hand. A `cp`-based fork
leaves those stale; a tiny "rename your theme" helper (or a note in the doc) would help. Forked theme
builds and serves fine (`Theme: odalys`, 12 pages, 0 broken links).
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
