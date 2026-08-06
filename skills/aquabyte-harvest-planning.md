---
name: Plan a harvest from biomass growth and harvest reports
description: Track average weight and weight distribution toward a target size, then pull the harvest report for packed and round weights, loss factor and superior rate — the smolt-to-harvest planning loop.
api: openapi/aquabyte-data-api-openapi.yml
operations:
  - get_sites_sites_get
  - get_biomass_daily_biomass_get
  - get_biomass_harvest_report_biomass_harvestReport_get
  - get_pen_welfare_data_welfareScores_get
  - get_pen_superior_rate_superiorRate_post
---

# Plan a harvest from biomass growth and harvest reports

## Before you start

- Base URL `https://api.aquabyte.ai/v3`, `apikey` header on every request, 1000 requests/hour.
- Resolve pens with `get_sites_sites_get` first; everything below keys on `penId`.

## Step 1 — track growth

```
GET /biomass?penId={penId}&fromDate=2026-01-01&toDate=2026-08-06&bucketSize=500
```

`get_biomass_daily_biomass_get` returns one `BiomassDailyModel` per pen-day:

- `avgWeight` — average weight in **grams**
- `kFactor` — condition factor
- `cv` — coefficient of variation, the spread of the population
- `sampleSize` — samples behind the estimate
- `weightDist` — a histogram as two parallel arrays, `interval` and `distribution`

`bucketSize` sets the histogram bucket in grams and defaults to 1000. Drop it to 500 or 250
when you are close to a grading boundary and need to see how much of the population sits
either side of it.

All measurement fields are nullable. A null `avgWeight` is a missing estimate, not zero —
never let one into a growth-rate calculation.

Fit growth on `avgWeight` over time to project the date a pen reaches target weight, and
carry `cv` alongside: a pen at target average with a wide distribution grades very
differently from a tight one.

## Step 2 — pull the harvest report

```
GET /biomass/harvestReport?penId={penId}&fromDate=2026-06-01&toDate=2026-12-31
```

`get_biomass_harvest_report_biomass_harvestReport_get` filters by **slaughter start date**,
not by report creation date. Each `BiomassHarvestReport` carries:

| Field | Use |
|---|---|
| `mainReport` | true for the authoritative report — a pen can carry several |
| `asOfDate` | latest data behind the report; matters for forecasts |
| `lastFeedingDate` | the starvation boundary |
| `slaughterStartDate` / `slaughterEndDate` | the slaughter window |
| `avgRoundWeightGrams` | average round weight at last feeding |
| `avgPackedWeightGrams` | average packed weight after processing |
| `lossFactor` | total loss through packing, excluding starvation |
| `packingMethod` | `HOG`, `WFE`, or null for a custom loss |
| `measurementCount` | fish measured for the report |
| `coefficientOfVariation` | spread |
| `superiorRate` | share grading superior |
| `packedWeightDistribution` / `roundWeightDistribution` | objects keyed by kg interval |

Filter to `mainReport: true` unless you are deliberately comparing scenarios. `fishType` is
null on any report created before 2025-08-14 — treat it as unknown, not as absent.

Round versus packed is the distinction that drives the money: `avgRoundWeightGrams` is the
fish at last feeding, `avgPackedWeightGrams` is what leaves the plant, and `lossFactor` and
`packingMethod` explain the gap.

## Step 3 — check quality before you commit

Downgrades destroy harvest value, so read welfare alongside weight:

```
GET /welfareScores?penId={penId}&fromDate=2026-06-01&toDate=2026-08-06
```

`get_pen_welfare_data_welfareScores_get` returns 17 indicators — `bodyWound`, `scaleLoss`,
`snoutWound`, `maturation`, `eyeBleeding`, `eyeClouding`, `exophthalmos`, `opercularDamage`,
`backDeformity`, the four fin categories, jaw deformities, `mechHeadWound`. Each is a
`WelfareScoreDetail` with `active` and `healed` proportions at severity 1, 2 and 3, plus
`nothing` (the healthy share) and its own `sampleSize`. Every indicator is nullable; a null
means not scored.

A rising `active` proportion at severity 2-3 on wound indicators is the signal that a
harvest window is closing.

## Step 4 — optional: the experimental superior-rate preview

```
POST /superiorRate?penId={penId}&fromDate=2026-06-01&toDate=2026-08-06
```

`get_pen_superior_rate_superiorRate_post` is the one non-GET operation on the API. It still
performs a read and still takes all its inputs as query parameters — there is no request
body. Each `SuperiorRateRecord` carries `superiorRate` and a `dataQuality` flag of `high`,
`medium`, `low` or empty; drop anything below `high` before acting on it.

Aquabyte marks this operation **experimental** in its own summary: "this is a preview
version of superior rate API. The details are subject to change." Do not build a production
dependency on its shape. The `superiorRate` field on the harvest report is the stable one.

## Handling failures

- **422** — `detail[].loc` names the bad parameter. Biomass and harvest reports take
  `fromDate`/`toDate`, never `fromTime`/`toTime`.
- **401** — bad or missing `apikey`.
- Cursor: `/biomass` and `/welfareScores` return `nextToken` past 10,000 records; repeat the
  request with `&nextToken=` until it is absent. `/biomass/harvestReport` and
  `/superiorRate` are not paginated.
