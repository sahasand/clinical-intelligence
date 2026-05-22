# Clinical Intelligence

A standalone single-file HTML dashboard for clinical-trial research, built directly against the public **ClinicalTrials.gov API v2**. No backend, no build step, no framework. Open `index.html` in any modern browser and it works.

Designed as a daily-driver tool for a Director of Data Science at a CRO — competitive intelligence, protocol design support, feasibility, and site selection.

---

## Architecture

- **Single file**: everything lives in `index.html` (HTML + CSS + JS in one document).
- **No backend**: the browser calls `https://clinicaltrials.gov/api/v2/studies` directly via `fetch`.
- **CDN dependencies only**:
  - **ApexCharts 3.49** — all charting.
  - **SheetJS (xlsx) 0.18** — CSV export from any table.
  - **Google Fonts: Inter + JetBrains Mono** — typography.
- **Design language**: dark glassmorphic shell, teal (#2dd4bf) accents, layered radial gradients, frosted-glass cards (`backdrop-filter: blur`), Inter 14px base.

## Features (6 tabs)

| # | Tab | What it does |
|---|---|---|
| 01 | **Trial Search** | Multi-field search (condition, intervention, sponsor, NCT, phase, status, date range) with paginated results table. Row click → detail modal. |
| 02 | **Sponsor Pipeline** | Pulls all trials for a sponsor, renders KPIs (total/active/recruiting/completed/areas), stacked bar by phase×status, top therapeutic areas, and full pipeline table. |
| 03 | **Endpoint Benchmarking** | Pulls all trials for a condition, extracts and **normalizes** primary endpoints, ranks them, and breaks them down by phase. Answers *"what endpoint is standard for my indication?"* |
| 04 | **Investigator Finder** | Searches by condition + optional country/state/city. Aggregates **PIs** (from `overallOfficials` and per-site `contacts` with `PRINCIPAL` role) and **sites**, ranks by trial count, plus country donut. |
| 05 | **Eligibility Analyzer** | Aggregates min/max age, sex, healthy-volunteer flag, and top eligibility-text keywords across all trials in a condition. |
| 06 | **About** | Architecture and feature reference. |

All tables support: column **sort**, row **click → detail modal**, and one-click **CSV export** via SheetJS.

### Detail modal
Fetched on demand from `GET /studies/{nctId}`. Shows full title, phase/status badges, sponsor, enrollment, all dates, design (allocation/masking/model/purpose), brief summary, conditions, arms/interventions, primary + secondary endpoints (with time frames), eligibility (parsed fields + raw criteria block), overall officials, and all locations with PI names + facility status.

## CT.gov API v2 — query patterns used

Base: `https://clinicaltrials.gov/api/v2/studies`

| Param | Used for |
|---|---|
| `query.cond` | Condition search |
| `query.intr` | Intervention search |
| `query.spons` | Sponsor search |
| `query.term` | NCT ID or free text |
| `query.locn` | Concatenated `country, state, city` filter |
| `filter.overallStatus` | Status filter |
| `filter.advanced` | Phase (`AREA[Phase]PHASE2`) and date ranges (`AREA[StartDate]RANGE[from,to]`) |
| `pageSize` | 50 for paged search, 200 for bulk aggregation |
| `pageToken` | Cursor pagination |
| `countTotal=true` | Returns `totalCount` |

Single-study fetch: `GET /studies/{nctId}` for the detail modal.

### Bulk-pull helper
`ctgFetchAll(params, maxStudies)` walks `nextPageToken` up to 1000 studies (browser-side cap) at `pageSize=200`. Used by Pipeline, Endpoints, Investigators, and Eligibility tabs.

## Data shape

The `extractStudy(study)` helper flattens the deeply nested `protocolSection` payload into a single object exposing: `nctId`, `briefTitle`, `officialTitle`, `leadSponsor`, `sponsorClass`, `overallStatus`, `phases`, `enrollment`, `startDate`/`primaryCompletionDate`/`completionDate` structs, `conditions`, design fields, `arms`, `interventions`, `primaryOutcomes`/`secondaryOutcomes`, eligibility fields (`minAge`, `maxAge`, `sex`, `healthyVolunteers`, `eligibilityCriteria`), `locations`, `overallOfficials`.

## Endpoint normalization

`normalizeEndpoint(s)` lowercases, strips punctuation, collapses whitespace. This collapses *"Change from Baseline in HbA1c at 12 weeks"* and *"Change From Baseline in HbA1c at 12 Weeks"* to the same bucket while preserving the original label for display.

## Age parsing

`parseAge(str)` handles CT.gov age strings like `"18 Years"`, `"6 Months"`, `"2 Weeks"` and returns years. `categorizeAge(yrs)` buckets into Infant / Child / Adolescent / Adult / Older Adult.

## Files

```
clinical-intelligence/
├── index.html      Standalone dashboard (everything is here)
└── CLAUDE.md       This file
```

No package.json, no build, no node_modules. Deploy by uploading `index.html` to any static host (Vercel, Netlify, S3, GitHub Pages, file://).

## Running

```
open index.html
```

Or serve over any static server. There is no env config — the API is public and unauthenticated.

## Known limits

- **Browser-side max**: 1000 studies per query (CT.gov + sanity cap). Very broad conditions like "cancer" will hit this; narrow your query.
- **No persistent cache**: each query hits the live API. A session req counter is shown in the footer.
- **PI extraction**: only counts people with role containing `PRINCIPAL`, plus everyone in `overallOfficials`. CT.gov data quality on PI names varies.
- **No map**: investigator/site geography is shown as a country donut + table; adding a real map would require a tiles dependency (Leaflet or similar).

## Extending

- New tab: add a `<button class="tab">` + `<section class="panel">` and one async handler. Reuse `ctgFetch`, `ctgFetchAll`, `extractStudy`, `renderTable`, `exportRows`, and the shared badge / spinner helpers.
- New chart: ApexCharts `new ApexCharts(el, opts).render()` — palette is `CHART_COLORS` at top of script.
- New column: extend the `columns` array passed to `renderTable` — `{ label, val, render?, sortVal? }`.
