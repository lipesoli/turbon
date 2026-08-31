# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-page B2B sales dashboard ("Dellamed · TurbOn Canal Online B2B") delivered as one static, self-contained HTML file: `index.html`. There is no backend, no build step, and no package manager — the entire app (markup, CSS, data, and logic) lives in this one file (~3,000 lines / ~1.3MB), including large embedded JavaScript data literals with the actual sales figures.

## Development commands

There is no `package.json`, build tool, linter, or test suite in this repo — none exist to run.

- **Preview locally**: open `index.html` directly in a browser, or serve it (needed if you hit `file://` restrictions):
  ```
  python3 -m http.server 8000
  ```
  then visit `http://localhost:8000/index.html`.
- **External dependency**: Chart.js is loaded from a CDN (`jsdelivr`) at the bottom of the file, asynchronously, with `onload="_onChartLoad()"`. Charts degrade gracefully (functions check `typeof Chart==="undefined"` and bail) if the CDN is unreachable — keep that guard when touching chart code.
- **Verifying a change**: since there's no test suite, validate by opening the file in a browser and clicking through the affected nav item / filters / period buttons, and checking the console for JS errors.

## Architecture

The file has three regions in this order: `<style>` (design tokens + layout, lines ~11–229), the `<body>` markup (sidebar + page panels, ~231–597), then two `<script>` blocks (data + logic, ~598–3065).

### Layout & navigation

- `.sb` (sidebar): logo, nav items (`.ni`, `onclick="nav('<id>',this)"`), period picker (month/quarter/YTD buttons), and three custom multi-select filter widgets (Representante, Cliente, Produto).
- `.ct` (main content): a `.hdr` header plus five page panels, each `<div class="pnl" id="p-<id>">`, only one visible at a time (`.pnl.active`, toggled by `nav()`):
  - `p-geral` — "War Room" overview
  - `p-cli` — per-client view
  - `p-prod` — per-product view
  - `p-rep` — per-sales-rep view
  - `p-resultado` — month result summary
- `nav(id, el)` swaps the active panel/nav-item and calls `render()`.

### Data layer (top of the first `<script>` block, ~line 599–641)

All business data is embedded as JS object/array literals assigned to short, terse global variable names — this is the part most likely to need updating and the part hardest to read directly (some lines are 100KB+ of minified JSON-like literal), so treat these as a data snapshot, not code to hand-edit character by character:

| Var | Meaning |
|---|---|
| `FAT` | Faturamento (revenue): `rep → cliente → subgrupo(produto) → ano → mes → valor` |
| `PED`, `PEDH_SG`, `PEDH_SG_Q` | Pedidos pendentes (open orders), various group-bys |
| `FQ`, `SGQ`, `SQ`, `CQ`, `PEDH_SG_Q` | Quarterly-aggregated counterparts of the monthly data |
| `SG`, `SS` | Sell-out / secondary sales data |
| `MR` | Meta (target) por representante, por mês |
| `MC` | Meta por cliente |
| `CG`, `CS`, `CD` | Carteira (backlog/pipeline) by grupo / subgrupo / data |
| `GT`, `ST`, `GST`, `SST` | "Top item" lookups per grupo/subgrupo (best-selling product per client, etc.) |
| `AG`, `AS`, `CI`, `CP`, `SB`, `SP`, `RP` | Lookup lists/arrays (all clients, all products, client subsets, etc.) used to populate filters/iterate |
| `EST`, `EST_SG` | Estoque (stock) by item / by subgrupo |
| `ITEM_SG` | Item → subgrupo (product → product-family) mapping |
| `G2R` | Grupo (cliente) → Representante mapping — used constantly to filter by rep when a client filter is active |
| `MK`, `NA`/`NI`, `NSA`/`NST` | Scalar constants (markup factor, counts) |
| `CURR_MES` | Current reference month |
| `MN_PT` | Month-name lookup (`['','Jan','Fev',...]`) |

The filter `<div class="cf-opt">` option lists in the sidebar HTML (Representante/Cliente/Produto) are a separate, hand-listed source of truth for the dropdown UI and must stay in sync with the actual values used as keys in the data vars above (e.g. `AG`/`G2R` keys) when adding/removing a client or rep.

### Query helpers (`qFP`, `qFP_PED`, `qFP_PEDH`, `qSG`, `qSS`, `qSQ`, `mR`, `crt`, …)

Generic aggregators that walk the nested data objects given optional filters — `fR` (rep), `fC` (cliente/grupo), `fS` (subgrupo/produto), a year `a`, and an array of months `ms` — and sum the matching leaf values. Almost every rendering function calls one of these rather than touching `FAT`/`PED`/etc. directly; add new aggregations here rather than inlining new traversal logic in a render function.

### Rendering pipeline

`render()` (~line 775) is the single entry point that re-draws the currently active panel; it's called after navigation, after a filter changes, and after the period (month/quarter/YTD) changes. It dispatches to per-section render functions named `r<Section>` (e.g. `rGeralEvol`, `rTopCli`, `rRuptura`, `rConcentracao`, `rCliMain`, `rProdMain`, `rRepMain`, `rResultMain`, `buildCartCards`/`buildCartTable`). Each owns one card/chart/table and writes into a specific element id via `innerHTML`. When adding a new metric or card, follow this same pattern: one `r<Name>` function, called from `render()`, guarded with `if(!el)return;` after the `$(id)` lookup.

### Filter & period state

- `SEL` — array of selected months; `CH` — live Chart.js instance registry (keyed by canvas id, destroyed/recreated on re-render).
- `CF = {rep:[], cli:[], sg:[]}` — currently selected filter values for the three custom multi-select dropdowns.
- `cfOpen/cfSearch/cfClick/cfUpdateUI/cfRemove/clrFilters` manage those dropdowns; `clickMes/setP/syncPeriodButtons` manage the month/quarter/YTD period picker.

### Formatting helpers

`$(id)` (getElementById shorthand), `S(id,v)`/`H(id,v)` (set textContent/innerHTML by id), `M(v)` (compact BRL currency: `R$1.2M`/`R$45k`), `P(v,b)` (percentage), `mkLabel(mk)` (`"2026-08"` → `"Ago/26"`).

## Conventions

- Code is plain ES5-style JS (`var`, `function`, no modules, no arrow functions) — match this style rather than introducing `const`/`let`/arrow functions/classes.
- CSS uses semantic custom properties for status colors: `--grn`/`--grn2`/`--grn3` (good), `--red`/`--red2`/`--red3` (bad), `--blu`/`--amb` families for info/warning, each with a base/light-bg/dark-text triplet — reuse these instead of hardcoding new colors.
- Git history is a flat sequence of "Update index.html" commits (this file is periodically regenerated by pasting a fresh data export into the `var FAT/PED/...` literals). When making a real code change, write a descriptive commit message instead of continuing that pattern.
