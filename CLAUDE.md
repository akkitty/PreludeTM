# CLAUDE.md — PreludeTM

> Maintenance reference for an app that already exists. Describes what's
> actually in the codebase today and the gotchas that bit during the build.

---

## 1. What this app is

A single-page tool for dealing a *Terraforming Mars: Prelude* draft. Pick a
player count (1–5), tap Roll, and each player gets 4 cards dealt from the
35-card Prelude deck with no repeats across players — mirroring the real
setup rule (each player picks 2 of their 4). Built for one person's phone use
at the table, not a general audience.

Live at `https://akkitty.github.io/PreludeTM/`, served by GitHub Pages from
`github.com/akkitty/PreludeTM` (public repo, `master` branch, root folder).

---

## 2. Current architecture

Everything is one file: `index.html`. No build step, no framework, no
`package.json`, no dependencies. The 35 base Prelude cards (`P01`–`P35`) are
embedded inline as a `CARDS` JS array (`num`, `name`, `file`), each pointing
at `assets/cards/<num>-<slug>.jpg`.

### Roll flow
`roll()` shuffles `CARDS`, slices `CARDS_PER_PLAYER` (4) cards per player off
the top with no replacement, renders a `.player` section per hand into
`#results`, and stashes the dealt hands in module-scope `lastHands` (array of
arrays of card objects) for the share feature to reuse. Tapping a card opens
it full-size in a `<dialog>` lightbox.

### Share feature
The "Share each hand as an image" button (`shareBtn`) renders **one canvas
image per player** — not one combined image — via `buildPlayerImage(index)`:
a colored-background header ("PreludeTM Draft" / date / "PLAYER N") over a
2×2 grid of that player's cards, drawn at `cardW = 640` canvas px per card
(~1380×1104px, ~470KB JPEG per player). Background color cycles through the
5-entry `PLAYER_COLORS` palette by player index.

All players' images are built with `buildAllPlayerImages()` and handed to
`navigator.share({ files })` in one call when the Web Share API supports
multiple files (true on Android Chrome, which is the actual target device) —
this opens the native share sheet with all images attached at once. Falls
back to sequential triggered downloads (staggered via `setTimeout`) when
`navigator.canShare` is unavailable or returns false, e.g. desktop browsers;
some desktop browsers block downloads after the first one triggered without
a fresh user gesture, and that's not currently worked around since it's not
the primary use case.

### Cache-busting
`ASSET_VERSION` (currently `'v2'`) is appended as `?v=...` to every image
`src`. **Bump this string any time `assets/cards/*.jpg` content changes**,
even if filenames stay the same — GitHub Pages' CDN and mobile browsers both
cache images aggressively, and without a version bump a phone that already
loaded the app once will keep silently serving stale image bytes after a
redeploy. This is exactly what happened once already (see §5).

---

## 3. Card data and images

Source: [hadronikle/Complete-Terraforming-Mars-Card-Database](https://github.com/hadronikle/Complete-Terraforming-Mars-Card-Database)
(`index.html`, a `const CARDS = [...]` array with `cat`, `exp`, `tags`,
`num`, `name`, `img` (full-res), `thumb` (200×142)).

**The filter to get exactly the 35 base Prelude cards is narrower than it
looks.** The naive `exp === 'Prelude'` or `cat === 'Prelude'` filters each
independently over-match: the same upstream file also carries "Prelude 2"
and Venus Next promo-prelude cards that share one of those two fields with
the base set (their `cat` is generically `"Prelude"` even though `exp` is
`"Prelude 2"`, or vice versa depending on how the source has been updated —
this actually changed between two fetches on the same day during the build).
The correct filter, verified to land on exactly `P01`–`P35`:

```js
cards.filter(c => c.cat === 'Prelude' && c.exp === 'Prelude' && !c.tags.includes('Promo'))
```

**Images were regenerated once, deliberately, away from the source repo's
defaults.** The upstream `thumb` field (200×142) was used initially and
turned out too low-res once viewed on a phone; the upstream `img` field
(2100×1500, ~1MB) was too heavy. `assets/cards/*.jpg` were produced with a
one-off Node + `sharp` script: download each `img`, resize to 900px wide,
re-encode as JPEG quality 82. That resize script wasn't kept in the repo —
if the images ever need regenerating, rebuild it the same way (fetch → sharp
`.resize({width: 900}).jpeg({quality: 82})`), and remember to bump
`ASSET_VERSION` afterward.

---

## 4. How to change things

No dev server needed for most edits — `index.html` is self-contained and can
be opened directly, or served with anything static (e.g. `npx serve .`) for
a closer-to-production check. To ship a change:

```bash
git add -A
git commit -m "..."
git push        # master → GitHub Pages redeploys automatically, usually within ~1 min
```

Common changes:
- **Card pool / count**: edit the `CARDS` array and `assets/cards/`. Keep
  `CARDS_PER_PLAYER` and the P01–P35-style `num` scheme consistent if you
  add cards.
- **Max players**: `MAX_PLAYERS = Math.min(5, Math.floor(CARDS.length / CARDS_PER_PLAYER))`
  — the `5` is a deliberate cap matching the physical game's player limit,
  not a limit imposed by card math (card math alone allows 8 with 35 cards).
- **Player colors**: `PLAYER_COLORS` array, indexed by player number, cycles
  if there are ever more entries than colors.
- **Share image layout/resolution**: all in `buildPlayerImage()` — `cardW`
  controls both the on-image card size and the effective sharpness after
  whatever the receiving app (Messages, WhatsApp, MMS) recompresses.

---

## 5. Known limitations and history worth knowing

- **Low-res share images (fixed once already).** The first share-image
  implementation drew cards at `cardW = 340` even though the source images
  were 900px wide — real resolution loss on top of messaging-app
  recompression. Fixed by raising `cardW` to 640. If "the shared image looks
  blurry" comes up again, check `cardW` in `buildPlayerImage()` before
  assuming it's a transport/compression issue.
- **Stale image cache (fixed once already, mitigated going forward).** Early
  on, a phone that had loaded the app before an image update kept some old
  low-res images cached inconsistently with new ones. `ASSET_VERSION`
  cache-busting was added specifically to prevent a repeat — see §2.
- **Desktop share fallback is untested/imperfect.** Multiple
  programmatically-triggered downloads in one click can get blocked by some
  browsers after the first. Not fixed, since the app's only real user is on
  Android Chrome, where `navigator.share` with multiple files works.
- **No offline/PWA manifest.** The app works offline after first load only
  because the browser's own HTTP cache still has the assets — there's no
  service worker or manifest making that a guarantee.
- **No tests, no CI.** Appropriately sized for a single-file personal tool;
  verification has been manual (local static server + Chrome) each time.
