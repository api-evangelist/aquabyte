---
name: Mirror Aquabyte pen data into your own database
description: Walk every site and pen a key can see, then bulk-pull biomass, lice, welfare, behaviour and environmental time series with penId=all and nextToken cursor pagination — the flow Aquabyte built v3.1 for.
api: openapi/aquabyte-data-api-openapi.yml
operations:
  - get_sites_sites_get
  - get_biomass_daily_biomass_get
  - get_pen_lice_count_liceCount_get
  - get_pen_welfare_data_welfareScores_get
  - get_behavior_swim_speed_behaviour_swimSpeed_get
  - get_behavior_breathing_index_behaviour_breathingIndex_get
  - get_environmental_environmental_get
---

# Mirror Aquabyte pen data into your own database

Aquabyte's own release note names this as the reason v3.1 exists: "One of the main use case
is for users to download Aquabyte data into their own database."

## Before you start

- Base URL: `https://api.aquabyte.ai/v3`
- Auth: send your issued key as the `apikey` header on **every** request. Nothing else is
  accepted; a missing or bad key returns `401` with an empty body.
- Budget: **1000 requests/hour**. There are no rate-limit response headers, so count your own
  calls — you cannot read remaining quota from a response.
- Use the **v3.1 query-parameter forms**. The `/pens/{penId}/...` path forms are marked
  `deprecated: true` in the specification.

## Step 1 — establish the inventory

Call `get_sites_sites_get`:

```
GET /sites
apikey: {API_KEY}
```

The response is `{"sites": [...]}`. Each `Site` carries `id`, `name`,
`governmentSiteNumber`, an optional `external_site_id` (your own system's id, if the
integration was configured), and a `pens` array. Each `Pen` carries `id`, `name`, `penCode`,
`isActive` and an optional `external_id`.

Persist sites and pens first — every other record joins back on `penId`, and this is the
only operation that resolves an opaque pen id to a human name. `/sites` is not paginated.

## Step 2 — bulk-pull each time series with penId=all

Do **not** loop per pen. Pass `penId=all` once per resource and let the server fan out:

```
GET /biomass?penId=all&fromDate=2026-01-01&toDate=2026-01-31
```

Run the same pattern for each series:

| Resource | Operation | Window params |
|---|---|---|
| Biomass | `get_biomass_daily_biomass_get` | `fromDate` / `toDate` |
| Sea lice | `get_pen_lice_count_liceCount_get` | `fromDate` / `toDate` |
| Welfare | `get_pen_welfare_data_welfareScores_get` | `fromDate` / `toDate` |
| Swim speed & tilt | `get_behavior_swim_speed_behaviour_swimSpeed_get` | `fromTime` / `toTime` |
| Breathing index | `get_behavior_breathing_index_behaviour_breathingIndex_get` | `fromTime` / `toTime` |
| Environmental | `get_environmental_environmental_get` | `fromTime` / `toTime` |

Daily-grain resources take **dates**; sub-daily resources take **times**. Mixing them is the
most common cause of a `422`.

Optional shaping:
- `bucketSize` on `/biomass` sets the `weightDist` histogram bucket in grams (default 1000).
- `period` on `/environmental` accepts `15min`, `h` or `D` (default `D`).
- `period` on `/behaviour/swimSpeed` accepts `h` or `D` (default `D`).

## Step 3 — drain the cursor

Each response is capped at **10,000 records**. If more exist, the body carries a
`nextToken`. Repeat the identical request with that token appended:

```
GET /biomass?penId=all&fromDate=2026-01-01&toDate=2026-01-31&nextToken=(A_TOKEN)
```

Keep going until `nextToken` is absent or null — that means the result set is complete.

Do not change `fromDate`/`toDate` or any other parameter while draining a cursor. Keep every
parameter byte-identical and vary only `nextToken`.

## Step 4 — write it down

Each collection arrives under a key named for the resource — `biomass`, `liceCount`,
`welfareScores`, `swimSpeed`, `breathingIndex`, `data` for environmental. There is no shared
envelope, so unwrap per resource.

Natural primary keys:
- daily series: `(penId, date)`
- period series: `(penId, fromTime, toTime)`

Both are stable, so upsert on them and re-running a window is safe.

## Handling failures

- **422** — a parameter is wrong. `detail[].loc` names it, e.g. `["query","fromDate"]`.
  Fix the parameter; retrying the same request will fail again.
- **401** — the `apikey` header is missing or invalid. Do not retry.
- **429 / 5xx** — neither is declared in the specification. Treat any non-200 that is not
  422 or 401 as transient, back off exponentially, and stay inside the 1000/hour budget.

## What this API will not do

It is read-only and poll-only. There are no webhooks, no callbacks, no streaming and no
AsyncAPI document, so there is no way to be notified of new data — schedule a poll instead.
Daily series are the natural cadence; a nightly incremental pull of the last few days
covers late-arriving measurements.
