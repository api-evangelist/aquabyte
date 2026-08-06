---
name: Monitor sea-lice burden across a farming licence
description: Resolve the pens under a site, pull per-pen sea-lice counts with the regulatory converted variants, and read them against environmental temperature — the weekly lice-reporting loop in Norwegian salmon farming.
api: openapi/aquabyte-data-api-openapi.yml
operations:
  - get_sites_sites_get
  - get_sites_siteId_sites__siteId__get
  - get_pen_lice_count_liceCount_get
  - get_environmental_environmental_get
---

# Monitor sea-lice burden across a farming licence

Sea-lice counting is Aquabyte's most regulated data product. This skill covers reading it
correctly, including the distinction between raw and converted counts that trips up most
first integrations.

## Before you start

- Base URL `https://api.aquabyte.ai/v3`, `apikey` header on every request.
- Budget 1000 requests/hour.
- Use the query-parameter forms. `GET /pens/{penId}/liceCount` still routes but is
  `deprecated: true`.

## Step 1 — find the site and its pens

If you know the site:

```
GET /sites/{siteId}
apikey: {API_KEY}
```

`get_sites_siteId_sites__siteId__get` returns the `Site` with its `pens` array. The
`governmentSiteNumber` field is the official locality number — use it to join Aquabyte data
to the public reporting record. If you do not know the site id, call `get_sites_sites_get`
and search by `name` or `governmentSiteNumber`.

Filter to `isActive: true` pens before counting anything; inactive pens will return empty
series and waste requests.

## Step 2 — pull the lice counts

```
GET /liceCount?penId={penId}&fromDate=2026-07-01&toDate=2026-08-06
```

`penId` accepts a single pen, a comma-separated list, or `all`. For one licence, pass the
pen list from step 1 in a single call rather than looping.

Each `LiceCount` row is one pen-day:

| Field | Meaning |
|---|---|
| `sampleSize` | number of samples behind the estimate |
| `adultFemale` | average adult female lice per fish |
| `adultFemaleConverted` | the **converted** adult female figure |
| `mobile` | average mobile lice per fish |
| `mobileConverted` | the **converted** mobile figure |
| `caligus` | average *Caligus* lice per fish |

**Read the converted fields for anything that compares to a regulatory threshold.** The raw
and converted values are different numbers and they are not interchangeable. Every count
field is nullable — a null means no estimate for that pen-day, which is not the same as a
count of zero, and averaging nulls as zeros will understate burden.

Always carry `sampleSize` through to whatever you build. A count from a handful of samples
is not a count from thousands, and the API gives you no other confidence signal.

## Step 3 — read it against temperature

Lice development is temperature-driven, so pull the matching environmental window:

```
GET /environmental?penId={penId}&fromTime=2026-07-01T00:00:00&toTime=2026-08-06T00:00:00&period=D
```

`get_environmental_environmental_get` returns `temperatureAvg` in Celsius per pen-period,
alongside `oxygenPct`, `salinity`, `cameraDepthAvg` and `fishDensity`. With `period=D` the
grain lines up one-to-one with the daily lice rows and you can join on `penId` + date.

Note the parameter difference: lice takes `fromDate`/`toDate`, environmental takes
`fromTime`/`toTime`. Passing a date where a time is expected returns a `422`.

## Step 4 — paginate if the window is wide

Both operations cap at 10,000 records and return a `nextToken` when there is more. Repeat
the request with `&nextToken=(A_TOKEN)` and keep every other parameter identical until
`nextToken` is gone.

## Handling failures

- **422** — check `detail[].loc`. Usually `fromDate` vs `fromTime`, or an unknown `penId`.
- **401** — bad or missing `apikey`.
- Camera depth matters for interpretation: lice distribute by depth, so a series where
  `cameraDepthAvg` moved is not measuring the same thing over time. Pull it and keep it.
