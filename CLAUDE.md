# CLAUDE.md

Guidance for AI assistants (and humans) working in this repository.

## What this is

**AgiS — カードスキャン (AgiS Card Scan)** is a mobile-first PWA for
**in-store trading-card appraisal**. A shop buyer photographs a Pokémon card
on their phone and, within a few seconds, gets:

1. **Card identification** — read the collector number (型式, e.g. `223/193`)
   from the photo via OCR, or type the card name.
2. **Market prices** — the raw ("素体"/sotai) and PSA10 graded price for that
   card, looked up from a pre-baked price cache (`prices.json`).
3. **Centering assist** — a draggable overlay on the photo that measures
   left/right and top/bottom border ratios and gives a rough PSA-grade
   estimate. **It never makes the final call — a human does.**

The guiding principle (see `設計_v0_MVP.md`) is to **reuse existing
infrastructure rather than build new tech**: card ID + pricing is an extension
of an existing market-data pipeline (AgiMa / GR), not new ML.

This is currently a personal tool for one operator ("なおなお"), not a shipped
product. Scope discipline matters — MVP is card-ID + price; centering is an
*assist line only*; state/condition ML is explicitly deferred to a later phase.

## Repository layout

The whole app is four files at the repo root — there is no build system, no
package manager, and no framework.

| File | Role |
|------|------|
| `index.html` | The **entire app** — HTML, CSS, and vanilla JS in one file. This is the PWA. |
| `prices.json` | The price cache the app reads. `{updated, count, cards}` where each card is keyed by slug. |
| `gen_prices.py` | Offline generator that builds `prices.json` from Excel ledgers/reports. **Runs on the operator's Windows PC, not here.** |
| `設計_v0_MVP.md` | Japanese design doc (v0 / MVP). Source of truth for scope, phases, and rationale. |

## How the pieces connect

```
Excel ledger + AgiMa reports  ──(gen_prices.py, run on Windows)──▶  prices.json
                                                                          │
photo ──▶ OCR (Tesseract.js) ──▶ collector number ─┐                     │
card name typed ────────────────────────────────────┼──▶ slug ──▶ lookup ◀┘  ──▶ price shown
                                                     │
photo ──▶ canvas edge detection ──▶ draggable box ──▶ border ratios ──▶ PSA estimate
```

The **slug** is the join key. Both the generator and the app slugify the same
way: lowercase, collapse whitespace/slashes to `-`, collapse repeats, trim.
`gen_prices.py:slug()` and the `slug()` inside `index.html` must stay in sync.
Example: card number `M2 110/080` → key `m2-110-080`.

## `index.html` — the app

Single IIFE in a `<script>` tag. Vanilla JS only, ES5-style (`var`, function
expressions) for maximum mobile-browser compatibility. No dependencies are
bundled; the **only** external dependency is Tesseract.js, lazy-loaded from a
CDN *on first capture* so the app starts instantly and works offline for the
price-lookup path.

Key subsystems (all in the one script):

- **Camera** (`getUserMedia`) — rear camera, continuous autofocus when
  available. Falls back gracefully to name-search-only if the camera is denied.
- **Level / 水平器** — uses `deviceorientation` (gamma) first, falls back to
  `devicemotion` gravity. iOS requires a permission request inside a user
  gesture, so sensors are enabled on the first `pointerdown`. A rotating
  crosshair turns green when within 4° of level.
- **Capture → centering** (`capture`, `autoCenter`, `redraw`, `updateCenter`) —
  after the shutter, a gradient-projection pass estimates an 8-line box
  (green = card edge, cyan = art frame). The user **drags** the lines to
  correct it (auto-detection is unreliable on black-border/SAR cards). Border
  ratios drive `psaEst()`: within 55:45 on both axes ≈ PSA10 range.
- **Price search** (`search`, `showDetail`) — matches typed text against card
  names and slugs, shows multiple candidates sorted by PSA10 price, tap to see
  detail. **Never asserts a single value** — always points the operator to
  スニダン (Snkrdunk) actuals for the final price.
- **OCR** (`loadOCR`, `cropBottom`, `extractType`, `runOCR`) — after capture,
  crops the bottom ~20% of the clean original, contrast-stretches it, and reads
  only the collector number (e.g. `223/193`). If found and the input is empty,
  it auto-fills and searches.

### Conventions when editing `index.html`

- **Keep it one file, no build step.** Do not introduce a bundler, npm, or a
  framework unless explicitly asked — the "open a URL on a phone" simplicity is
  a deliberate design choice.
- **Stay ES5-compatible** in the script — match the existing `var` / function
  idiom. This runs on arbitrary in-store phones.
- **UI text is Japanese.** Keep it that way.
- **Never let the tool make the grading verdict.** Centering and PSA estimates
  are assists; the copy consistently says the human's eye is final. Preserve
  that framing.
- CSS uses the `--brand1/--brand2/--good/--warn/--bad` custom-property palette
  defined in `:root`. Reuse those tokens.
- Prices are fetched with a cache-buster: `fetch('prices.json?'+Date.now())`.

## `gen_prices.py` — the price generator

- Runs **offline on the operator's Windows machine** (note the hardcoded
  `C:\...` `DATA` and `OUT` paths). It is **not** part of the deployed app and
  is not run in this environment.
- Requires `openpyxl`. Reads the latest `*ledger*.xlsx` and
  `AgiMa相場レポート_*.xlsx` (picking the newest by an 8-digit date in the
  filename) and emits `prices.json`.
- **Precedence**: the ledger ("台帳") is authoritative and loaded first; report
  rows only fill gaps (a key already carrying a `psa10` is not overwritten —
  see `put()`).
- `src` on each card records provenance: `台帳`, `GR乖離`, `GR出来高`, `GR差分`.
- The `updated` date is currently **hardcoded** (`datetime.date(2026,7,3)`) —
  update it when regenerating.

To refresh prices you edit/run this script on the Windows box and commit the
resulting `prices.json`; there is no server-side regeneration.

## `prices.json` — the data contract

```json
{
  "updated": "2026-07-03",
  "count": 117,
  "cards": {
    "sv9-126-100": { "name": "リーリエのピッピ", "sotai": 39400, "psa10": 40950, "src": "台帳" }
  }
}
```

- Keyed by slug. `sotai` / `psa10` are yen integers or `null`.
- The app reads only `cards`; `updated`/`count` are informational.
- Written with `ensure_ascii=False, indent=0` (compact, UTF-8, Japanese kept
  literal). Preserve that if you hand-edit.

## Running / testing

There is no test suite and no server code. To run the app locally, serve the
directory over HTTP (camera/sensors need a secure context — `https` or
`localhost`):

```bash
python3 -m http.server 8000
# then open http://localhost:8000/ (camera works on localhost; real devices need HTTPS)
```

Camera, level sensors, and OCR only fully work on a real phone over HTTPS. On a
desktop you can still exercise the **name-search price lookup** path.

## Git workflow

- Active development branch for AI work: **`claude/claude-md-documentation-mavlte`**.
  Develop, commit, and push there; do not push to `main` without explicit
  permission.
- Commit messages in history are lowercase, imperative, and describe the
  behavior change concisely (e.g. "centering: draggable 8-line overlay…").
  Match that style.
- Push with `git push -u origin <branch-name>`.

## Scope guardrails (from the design doc)

When proposing changes, respect the MVP boundaries in `設計_v0_MVP.md`:

- **In scope (MVP):** photo → collector-number OCR → price display, plus the
  centering assist line.
- **Deferred (Phase 3+):** condition/state ML, image recognition of card
  quality, user management, legal/support — do not build these speculatively.
- Do not add a live/headless-browser price backend; the design deliberately
  chose a **pre-baked cache** so the app is fast and works on weak in-store
  connectivity.
