# Baseline Farmer — repair notes

Two files changed substantively (`guesser.html`, `baseline_farmer.html`), one trivially
(`market_strategy.html`). No assets touched. No features removed.

---

## 1. The Guesser page was silently destroying the shared database

**Severity: critical — permanent data loss, affected everyone who used the site in the wrong order.**

`guesser.html` opened the shared database with an empty upgrade handler:

```js
const r = indexedDB.open(BF_DB, 2);
r.onupgradeneeded = () => {};     // creates the DB with ZERO object stores
```

`index.html` links straight to the Guesser, so a first-time visitor could easily land
there first. That created `baselineFarmerDB` at version 2 with no stores at all. The
dashboard then opens **that same version**, so its `onupgradeneeded` never fires and its
store-creation code never runs. From that moment every read and write threw
`NotFoundError`, forever, until the user manually cleared site data.

**Fixed.** `guesser.html` now creates the identical schema the dashboard expects, driven
by shared `BF_DB_VER` / `BF_STORES` constants so the two can't drift apart again. Both
files also detect a store-less database and rebuild it **at the same version** — never by
bumping, because the dashboard opens v2 explicitly and a higher version would make it
throw `VersionError`. Browsers already damaged by the old build heal themselves on next
visit.

## 2. The snapshot the Guesser page rendered was never written

**Severity: critical — this is the "never shows the snapshot" bug.**

`render()` read `node.features.refs`, `node.featureSnapshot.refs`, `node.athDist`,
`node.rangePos`, `node.virtualDev` and `node.caseText`. **None of those keys exist
anywhere in the project.** The dashboard writes its market read to `node.market` as
`{dist, rangePos, virtual, ...}` — different names, one level deeper.

Worse, two things were computed and then thrown away entirely:

- the **1H/4H/7D/30D/90D structure map** (`structure.map`), built in `renderStructure()`
  and used by `renderPrototype()`, was never attached to the node;
- **`caseText`**, the sentence explaining the call, same story.

`slimNode()` had no field for either, so even if they had been attached they'd have been
stripped on the way to storage. Result: five "No snapshot" tiles and an all-"—" evidence
panel on every machine, permanently, no matter how long the dashboard ran.

**Fixed on both sides.**

- Dashboard: added `refsFrom()` and `marketSnapshotFrom()`; the structure map now rides
  inside `node.market.refs`, so it inherits `slimNode`'s existing `market` passthrough.
  `caseText` is threaded through `featureSnapshot()` → `createSessionNode()` and added to
  `slimNode`. Reused nodes now refresh their snapshot and re-persist (previously a reused
  node's updated guess only reached disk incidentally).
- Guesser: new `snapshotOf()` normalizer reads the real shape and still tolerates the old
  and alternate ones, so **nodes already stored by the old build render correctly** rather
  than being orphaned. Timeframe tiles now colour by direction and show volume ratio;
  evidence gained price-path and hypothesis rows.

## 3. No reset, and the setup screen was unreachable

The setup overlay only appeared when `!portfolio.setupComplete`. But the dashboard's
`boot()` calls `initGuesserPortfolio(px, true, GUESSER_DEFAULTS)`, which sets
`setupComplete: true` on first load — so anyone who opened the dashboard first never saw
that overlay at all, and nothing could get them back to it.

**Fixed.** A **Reset** button on the Guesser stage opens a two-level dialog:

- **Restart the Guesser** — clears its portfolio, policy and nodes, then reopens the setup
  screen. Your dashboard numbers, journal and market cache are deliberately left alone.
- **Reset everything** — closes the connection, deletes the database, strips every
  `baselineFarmer*` localStorage key (including the `VisualTier` key, which nothing used
  to clear) and reloads. Genuine first-visit state.

The same **Reset everything** is also on the dashboard, next to Export/Import, behind a
confirm. Per your note I left `boot()`'s auto-create alone — changing it risks the
dashboard's own render path — and made the overlay reachable on demand instead.

---

## Also fixed

| | |
|---|---|
| **Setup wiped all training** | `applySetup()` wrote `weights:{}, humanWeights:{}, humanTrained:{}` to IndexedDB, silently erasing every learned weight and all journal training each time setup was applied. Now merges instead of overwriting. |
| **History timestamps all read "now"** | The table sorted and formatted by `x.t`, which `slimNode` never emitted; `Number(undefined) \|\| Date.now()` made every row show the current time and the sort a no-op. Now uses `created ?? t`, and `t` is persisted going forward. |
| **Sprite pinned to top tier** | `avatarTier()` divided by `Math.max(start,1)`, so a zero starting value turned `progress` into a raw dollar figure and forced `tier4`. Now returns `tier0` when there's no positive starting value, and clamps the score. |
| **Dead-end empty state** | The page was a pure reader; opening it before the dashboard showed "NO MARKET SNAPSHOT" with no way forward. It can now fetch the market itself (same endpoints and parsers as the dashboard, writing to the shared cache), plus an explanatory banner that distinguishes "no market data" from "market data but no decision recorded yet". |
| **Fragile IDB reads** | `loadAll()` let a single missing store reject the whole load. Now uses guarded per-store reads. |
| **Missing nav link** | `market_strategy.html` lacked the "How it works" link every other page has. |

## Verified

Round-trip test (dashboard `featureSnapshot` → `slimNode` → guesser `snapshotOf`): 18/18
checks pass, covering fresh nodes, legacy nodes written by the old build, and empty nodes.
Mock-IndexedDB test of the real `openDb` code: 11/11 checks across four scenarios —
Guesser-first, already-damaged browser healed from either page, and dashboard-first.
All inline scripts pass `node --check`; all local asset links resolve; every element ID
referenced from JS exists.

## Not changed, deliberately

- **`indexoldV.html`** (71 KB) and **`guesser_sprite_sheet_source.png`** (2.1 MB) are
  referenced by nothing. Left in place — they're yours to cut.
- **~60 lines of dead CSS** in `guesser.html` (`.rightArm`, `.farmerHat`, `.visorGlow`,
  `.headAnim` and the matching keyframes) target an inline-SVG avatar that no longer
  exists; the avatar is now a single div with a background sprite. Harmless, but it's
  exactly the residue that misleads the next automated pass.
- **The sprite atlas math is correct** — I checked because it looked wrong. Sheets are
  2160×960, a 9×4 grid of 240 px cells, scaled to 2880×1280 so each cell lands at 320 px.

## One thing to know

`guesser.html` now makes network requests (CoinPaprika + Coinbase) when the shared market
cache is empty or older than three minutes. If you'd rather it stay a pure reader, delete
the `refreshMarket` call at the bottom of `bootGuesserPage()`; the banner's manual "Fetch
market data" button will still work.

Both pages need to be served over `http://` or `https://` for those fetches — opening the
files directly via `file://` will block them (and IndexedDB behaves inconsistently there
too). Any static server works: `python3 -m http.server` from the project folder.

---

# Sprite rebuild (second pass)

## What was actually wrong

Not the art — the slicing. `guesser_sprite_sheet_source.png` is a **design poster**:
six tier rows, nine labelled state panels each, every panel holding two or three
small figures, with glowing rounded-rectangle borders between panels. The tier
PNGs were produced by cutting sheet-style artwork on a naive 9x4 grid, which
captured the panel borders and neighbouring figures along with the intended one.

Measured across all six sheets, **26 of 150 displayed frames were damaged**:

| tier | damaged frames |
|---|---|
| 1 | 3 |
| 2 | 3 |
| 3 | 1 |
| 4 | 1 |
| **5** | **15** |
| 6 | 3 |

- **HARVEST frame 1 was broken in every tier** — a vertical panel border plus a
  sliver of a figure, roughly a third of the ink of its neighbours. Harvest was
  configured as a 4-frame loop at 240 ms, so the Guesser visibly blanked once per
  cycle every time it harvested.
- **Tier 5 — the "HOLOGRAPHIC" top tier, the whole point of the progression** —
  had bright vertical bars cutting through 15 of its frames.
- Alignment was actually fine (centres within ±2 px, foot baseline 214–215 px).
  Figures filled only ~68% of cell height, so the character sat small in a large
  empty box.
- ~6.6% of every sheet was faint sub-alpha-60 halo left by background removal.

## What the rebuild does

`build_sprites.py` (included) regenerates all six sheets from the originals:

1. **Isolates the dominant figure per cell** — scores blobs on area, figure-like
   proportions, per-row width variation (a border bar is near-uniform, a figure
   isn't) and nearness to centre; then keeps only the widest contiguous ink run,
   so detached neighbour slices can't survive. Morphological closing runs first,
   otherwise a dim waistband splits the figure and you keep only the torso.
2. **Drops the broken harvest frame** and repacks the column to 3 frames.
   `SPRITE_FRAMES.harvest` is now `3` in `guesser.html`.
3. **Rebuilds tier 5** as a holographic recolour (luminance → cyan ramp + masked
   bloom) of **tier 4, which was nearly undamaged**. Cleaner and more consistent
   than trying to salvage 15 bar-sliced frames.
4. **Strips the halo** and normalises scale and baseline with **one transform per
   sheet**, and one horizontal offset per animation column — so motion between
   frames inside a state is preserved exactly rather than flattened.
5. **Writes at 2880×1280**, the exact size the CSS displays, so the browser blits
   1:1 instead of upscaling a 2160×960 sheet with its own filter.

## Result

Figures are ~1.3× larger in frame, centred, full-height, single. Verified
programmatically: **0 empty frames, 0 clipped frames, 0 off-centre frames** across
all 150 displayed frames.

**6.04 MB → 2.76 MB (54% smaller)** despite the higher resolution, via palette
quantisation.

## Honest limits

I repaired and recomposed your existing artwork — I did not draw new characters,
and I have no image generator in this session. Two things repair can't fix:

- **Detail ceiling.** The source figures are ~165 px tall; displayed at 320 px
  they're upscaled either way. They're cleaner and better-framed now, not sharper.
- **Tier 5 lost its distinct silhouette.** The original top tier was a smooth
  helmeted "ascended" figure — a real design progression. Mine is the tier-4
  farmer recoloured, so the silhouette no longer changes at the top tier. If you
  want that back, it needs regenerating rather than repairing.

If you do regenerate, the spec to match is: **9 columns × 4 rows, 320 px cells,
2880×1280, transparent background, one figure per cell, feet on a common baseline
at y≈285 within the cell, horizontally centred.** Column order: idle, thinking,
buy-core, buy-tactical, harvest, preserve, celebrate, setback, growth. Frames per
column: 4, 2, 3, 2, 3, 3, 3, 2, 3. Generate each frame individually — the failure
here came entirely from generating sheet-style images and slicing them afterwards.
