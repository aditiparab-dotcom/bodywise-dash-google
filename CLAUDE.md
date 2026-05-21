# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the dashboard

There is no build step. Open `index.html` directly in a browser:

```bash
open index.html
# or serve it locally
python3 -m http.server 8080
```

External dependencies load from CDN: Chart.js 4.4.1 and Google Fonts (DM Mono, Fraunces, Geist). No npm, no bundler.

## Architecture

Everything lives in a single `index.html` file with three sections: inline CSS in `<style>`, HTML structure in `<body>`, and all JavaScript in a single `<script>` block at the bottom.

### Data layer (lines ~511–3010)

All data is hardcoded as `const` arrays at the top of the script block — there is no backend or API:

| Constant | Contents |
|---|---|
| `RAW_DAILY` | 79 daily rows (Mar 1 – May 18, 2026): `{d, cost, impr, clicks, conv, val, ctr, cpm, cac, roas}` |
| `CAMPAIGNS` | ~15 campaign rows with spend, ROAS, CAC, verdict, notes |
| `SCORECARD` / `SCORECARD_TAGS` | Change-impact cards; `SCORECARD_TAGS` maps campaign names to product tags for filtering |
| `CHANGE_EVENTS` / `TIMELINE_TAGS` | Timeline entries; `TIMELINE_TAGS` maps event index → product tags |
| `VIDEO_DATA` | `{top, bad, watch}` arrays of video title performance rows |
| `FUNNEL_DATA` | Per-campaign PDP → ATC → Purchase funnel counts |
| `RATE_DATA` | Computed funnel rates + signal (`good`/`warn`/`bad`) per campaign |
| `YT_DATA` | YouTube-attributed conversion signals per campaign |
| `TAG_DAY_REAL` | Per-product-tag daily data for May 1–18, keyed by tag abbreviation |
| `TAG_DEFS` | Abbreviation → full name (`'HG' → 'Hair Gummies'`, etc.) |

When adding or updating data, all monetary values are in Indian Rupees (₹). The formatter `fmt()` inside `renderMetrics` auto-scales to L (lakh = 1e5) and Cr (crore = 1e7).

### Render flow

`render()` is the single entry point called by date filter changes and the Refresh button:

```
render()
  └── getFiltered()           → filters RAW_DAILY by [dateFrom, dateTo]
  ├── renderMetrics(data)     → top KPI cards (ROAS, spend, CAC, conversions, CTR)
  ├── renderCharts(data)      → ROAS trend + Cost vs Conv line charts + CAC chart
  ├── renderCampaignTable()   → sortable campaign table (uses full CAMPAIGNS, not date-filtered)
  ├── renderScorecard()       → change-impact scorecard (filtered by activeProductTag)
  ├── renderTimeline()        → two-column event timeline (filtered by date range + activeProductTag)
  ├── renderDynamicAnalysis() → contextual analysis text blocks keyed to date range
  └── renderProductTab()      → only if productTabInitialized (lazy, triggered by tab click)
```

Charts are stored in a `charts = {}` object. Always call `destroyChart(id)` before re-creating a Chart.js instance to avoid canvas conflicts.

### Tabs

Three tab panels toggled by `switchTab(id)`:

- `tab-perf` — Campaign Performance: table + scorecard + timeline
- `tab-product` — Product Tag Analysis: per-tag charts + day-on-day table (lazy-initialized)
- `tab-titles` — Video Title Performance: three sub-views (`top`/`bad`/`watch`) and three data views (funnel / rate / YT)

### Global state

```js
let dateFrom, dateTo         // active date range strings ('YYYY-MM-DD')
let activeProductTag         // currently selected product tag abbreviation, or null for ALL
let productTabInitialized    // lazy-init guard for product tab
let currentVideoView         // 'top' | 'bad' | 'watch'
let currentSort, sortDir     // campaign table sort state
```

### Thresholds (used for card colors and flag classes)

- ROAS: green ≥ 3×, yellow ≥ 2.5×, red < 2.5×
- CAC: green ≤ ₹400, yellow ≤ ₹500, red > ₹500
- CTR: green ≥ 0.8%, yellow ≥ 0.7%, red < 0.7%
- Funnel purchase rate: `flag-good` ≥ 4%, `flag-warn` ≥ 2%, `flag-bad` < 2%

## Business context

The "Cliff Week" preset (Apr 10–16) refers to a sharp ROAS collapse starting Apr 13, 2026 — from ~3.3× pre-cliff to ~2.5× post-cliff. Most analysis sections in the dashboard are structured around diagnosing and attributing this event. Campaign naming follows the convention `DG_<product>-<funnel stage>-<version>` (e.g. `DG_Body-Int-02` = Demand Gen, Body, Interest targeting, version 2).
