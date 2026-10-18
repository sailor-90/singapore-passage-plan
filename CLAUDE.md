# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Passage Plan Singapore · MT TITANIA. Offline PWA for building in-port passage plans in Singapore waters (anchorage/berth ↔ fairway). It computes waypoint courses, distances and ETAs, shows a vector chart of the port, and fills the ship's passage planning forms (Excel and PDF). UI is English; `README.md` is Indonesian.

Live: https://sailor-90.github.io/singapore-passage-plan/ (GitHub Pages, public repo).

## Sebelum mengubah kode

- Baca graphify-out/GRAPH_REPORT.md dulu untuk memahami struktur proyek. Folder itu hanya ada di komputer ini dan tidak ikut git; kalau tidak ada atau usang, jalankan /graphify dulu.
- Setelah perubahan yang menambah atau mengubah fungsi, fitur, atau file, dan hasilnya sudah dicek serta di-commit, jalankan /graphify . --update atau ingatkan user.

## Commands

No build, lint or test tooling (no package.json). Serve the folder statically and check in a browser:

```bash
python3 -m http.server 8765    # then open http://localhost:8765
```

- A local-only `.claude/launch.json` ("passage-plan", same command) lets the Claude desktop preview start it; it is git-excluded.
- Syntax check of the app script: copy lines from `<script>` at ~line 269 to the end into a temp file, replace every line longer than 20000 chars with `var X=null;`, then `node --check` it.
- Verifying exports in a browser: they go through `showResult(blob,name,kind)` (Excel/RTZ/CSV) and `showResults(list)` (PDF set). Override these on `window` in the console to inspect the blobs without downloading.
- The splash intro (`#intro`, runs once per session via `sessionStorage` key `sgIntro`) is skipped on reload. Clear that key to see it again. Browsers with `prefers-reduced-motion` show it static.

## Working with `index.html` (read this first)

Everything is one file by design (CSS, 4 pages, two bundled libraries, embedded data, app script). Never `Read` the whole file, and never `cat`/`grep` without `cut -c1-200`. Some lines are huge:

| What | Size |
|---|---|
| Chart SVG path lines (~lines 192–195) | 35K–636K chars each |
| Minified JSZip and pdf-lib (~lines 251, 266) | 97K / 524K chars |
| `const DATA=` port data (~line 270) | 521K chars |
| `const FORM_B64=` Excel template, base64 (~line 624) | 3.5 MB |
| `var TPL=` PDF templates, base64 (~line 953) | 630K chars |

Find them again with `awk 'length($0)>20000{print NR": "length($0)}' index.html`. Read ranges with `sed -n 'A,Bp' index.html | cut -c1-200`.

Edit with a small Python script that asserts the target string occurs exactly once, rather than the Edit tool on lines near the huge ones. Do not introduce a build step or split files without being asked.

Order in the file: CSS (`<style>`, includes the "Intro splash" block) → `#intro` splash + its inline script → `#app` with `<section class="page">` × 4 (`pgData`, `pgRoute`, `pgPlan`, `pgChart`) → libraries → app `<script>`.

Other files: `sw.js` (service worker, network-first with cache fallback so it works offline at sea), `manifest.webmanifest` and `*.png` (PWA icons), `README.md` (user-facing features and install steps).

## Architecture (app script)

- **Data**: `DATA` keys are `cal` (map calibration), `anc` anchorages, `fw` fairways, `pl` pilot/limit points, `tss`, `pbg` boarding grounds, `dg` dangerous-goods anchorages, `term` terminals, `aton` buoys/beacons, `net` (waypoint network: `nodes`, `edges`, `edgesAuto`, `lib` route library of ship passage plan files, `corr`), `berthDefault`. `NET` is `DATA.net`.
- **Route pipeline**: `rebuild()` (From/To change) → `buildRoute()` → `buildRoute0()` (a library variant via `libFor`/`buildLib`, or `netPath` over the network) → `applyNotes()` (user remarks saved in `sgWpNotes`) → `fillEnds()` (auto remarks, below). The current route is the global `R`; a user-edited copy lives in `edit` (`ensureEdit`, `edToR`). `ppRender()`/`apply()` redraw the plan cards and chart.
- **Calculation**: `rhumb(a,b)` is a Mercator rhumb line (R = 3440.065 NM). `ppCalc()` builds the plan rows (course, distance, leg speed, ETA). The Plan input `ppSpd` defaults to 10 kn in the markup (`planSpd()` only falls back to 8 if it is empty); per-leg speeds apply only when "speed per leg" is on (`useLeg()`). `P()`/`LL()` convert lat/lon ↔ SVG map coordinates using `DATA.cal`.
- **Ship data**: displacement comes from the `HYDRO` table by mean draft (`dispAt`). Air draught = `SHIP_HEIGHT` (32.3 m) − aft draft. `SHIP_HEIGHT` is declared twice in the script; keep both in sync.
- **Output**: `formData()` feeds both writers. Excel is written with JSZip into the `FORM_B64` template (ship's Form N 03); PDFs with pdf-lib onto `TPL` (passage plan, checklists N-03A/N-01/N-02, pilot card N-04); also RTZ and CSV export. From/To names in forms and file names come from the first/last waypoint name (`endShort`), so renaming WP1/last in the editor carries through; waypoint names themselves are not written into the Excel/PDF tables (positions and remarks are).
- **Editing**: the WP Edit sheet (`openEdit`) keeps the node `id` and exact position unless lat/lon change by more than 1e-5°. Routes the user saves go to `localStorage` `sgRoutes` (More → Save/Open), never into `DATA`.
- **Auto remarks by VHF control** (`CTLS`, `TERM_CTL`, `SECP`, `ctlOf`, `fillEnds`): WP1 gets "Departure Report to <control> VHF Ch. <n>", the last WP "Arrival Report to …", and mid-route sector changes "Entering <control> …". It only fills empty remarks (empty, the "posisi labuh" anchorage placeholder, or a bare `[draf]`). Terminals/anchorages are mapped by name first (Eastern anchorages → East Ch.12; Brani/Tanjong Pagar → Keppel Ch.05; Pasir Panjang, Jurong Port, Advario JI → Pasir Panjang Ch.18; JPUT, Horizon and Western anchorages → West Ch.68; Advario Helios → Jurong Ch.22), and mid-route WPs by approximate sector polygons measured from the SPI 2026 "Reporting Sectors" chartlet. "Entering Jurong Control" is skipped unless the route starts or ends there. These are the user's operating rules, not derived from code.
- **Persistence**: `localStorage` keys `sgShip`, `sgOfficers`, `sgRoutes`, `sgNetAdd`, `sgWpNotes`, `sgTab`, `sgBerthPos`, `sgPdfSel` via `lsGet`/`lsSet`. Form inputs use `pp*` ids (`ppShip`, `ppDate`, `ppEtd`, `ppFrom`, `ppTo`, `ppSpd`, …).
- Chart data is MPA PMC 02/2018 + PMC 03/2025, SPI Oct 2026. The tool is a planning aid and must be verified on corrected ENC/ECDIS, Notices to Mariners and Port Marine Notices (see README).

## Conventions and gotchas

- **Bump `VERSION` in `sw.js` whenever an app file changes** (currently `passage-plan-sg-v15`), otherwise installed phones keep the old copy. After a push, phones need two opens to pick up the new version.
- Navigation numbers (routes, waypoints, UKC, buoys, drafts, hydrostatics) are safety-critical. Do not change or "fix" them unless explicitly asked.
- Fonts load from Google Fonts (IBM Plex); offline the system fallback is used.
- `referensi/` (local folder with the MPA Singapore Port Information PDF and ship forms) is copyrighted and git-excluded; never commit it.

## Git workflow

- Public repo `sailor-90/singapore-passage-plan`. Claude commits locally; the user pushes with GitHub Desktop (Claude has no GitHub credentials).
- Commits must be authored as `sailor-90 <333464309+sailor-90@users.noreply.github.com>`. GitHub rejects the user's real Gmail (private email). After every commit, check `git log --format='%an <%ae>' -1` before telling the user to push.
- Local-only paths listed in `.gitignore` (and `.git/info/exclude`); never commit or push: `graphify-out/`, `referensi/`, `.claude/launch.json`, `.DS_Store`.
