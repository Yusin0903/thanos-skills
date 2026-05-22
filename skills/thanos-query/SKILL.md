---
name: thanos-query
description: >-
  Query Thanos HTTP API for cross-region Prometheus metrics via PromQL. Use when
  running PromQL/MetricsQL queries against a Thanos Querier or Query Frontend,
  discovering metrics/labels across regions, inspecting Thanos stores
  (sidecars/store-gateway), checking alerts/rules, troubleshooting empty query
  results, or analyzing time series data that spans multiple Prometheus
  instances. Triggers on: thanos query, promql, cross-region metrics,
  p50/p95/p99 latency, sidecar status, store-gateway, query frontend, label
  discovery across regions, empty results, missing data, data delay, no data
  found, query returns empty.
---

# Thanos Query

Query Thanos HTTP API directly via curl. Thanos implements the full Prometheus HTTP v1 API plus Thanos-specific endpoints (`/api/v1/stores`, dedup/partial_response params). Covers instant/range queries, label/series discovery, stores topology, alerts, and rules across multiple Prometheus backends.

## Environment

```bash
# $THANOS_METRICS_URL — base URL of Thanos Querier or Query Frontend HTTP API
#   Required. Set in ~/.claude/settings.json under `env`.
#   Example:  http://thanos-query.example.internal:10902
#             https://thanos.example.com
#
# $THANOS_AUTH_HEADER — optional full HTTP header line, if your deployment fronts
#   Thanos with a reverse proxy that requires auth (basic/bearer).
#   Example:  Authorization: Bearer <token>
#             Authorization: Basic <base64(user:pass)>
#   Leave empty/unset when Thanos is reached directly (Thanos has no built-in auth).
```

## Auth pattern

All curl commands use conditional auth — works for both reverse-proxied and direct deployments:

```bash
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/query?query=up" | jq .
```

When `THANOS_AUTH_HEADER` is empty, the `-H` flag is omitted automatically.

## Core endpoints

### Instant query

```bash
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=up' \
  "$THANOS_METRICS_URL/api/v1/query" | jq .

# Query at a specific time
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=up' \
  --data-urlencode 'time=2026-03-07T09:00:00Z' \
  "$THANOS_METRICS_URL/api/v1/query" | jq .
```

Params: `query` (required), `time` (RFC3339 or unix seconds), `timeout`, `dedup`, `partial_response`, `max_source_resolution`.

### Range query

```bash
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=rate(http_requests_total[5m])' \
  --data-urlencode 'start=2026-03-07T00:00:00Z' \
  --data-urlencode 'end=2026-03-07T12:00:00Z' \
  --data-urlencode 'step=5m' \
  "$THANOS_METRICS_URL/api/v1/query_range" | jq .
```

Params: `query`, `start`, `end`, `step` (required); plus `timeout`, `dedup`, `partial_response`, `max_source_resolution`.

### Label and series discovery

```bash
# All label names
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/labels" | jq '.data[]'

# Values of one label
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/label/region/values" | jq '.data[]'

# Find series matching a selector
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'match[]={job="api-server"}' \
  "$THANOS_METRICS_URL/api/v1/series?limit=20" | jq '.data'
```

### Thanos-specific: stores topology

```bash
# List all backing stores (sidecars, store-gateway, ruler, receive)
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/stores" | jq '.data | to_entries[] | {kind: .key, count: (.value | length)}'

# Detail: which stores per region, last-error state
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/stores" \
  | jq '.data | to_entries[] | .key as $k | .value[] | {kind:$k, name, lastError, labelSets}'
```

### Alerts and rules

```bash
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/alerts" | jq '.data.alerts[]'

curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/rules" | jq '.data.groups[]'
```

## Cross-region queries

Thanos exposes each backend Prometheus's `external_labels` on every metric. Common label conventions:

- `region` or `thanos_region` — the geographic / logical region
- `cluster` — K8s cluster name
- `environment` — env tier (dev/staging/prod)
- `replica` — HA Prometheus replica (deduped by querier)

Use these to scope queries:

```bash
# Filter by region label
--data-urlencode 'query=up{region="us-west"}'

# Group by region for cross-region comparison
--data-urlencode 'query=count(up) by (region)'

# P95 latency per region per HTTP verb
--data-urlencode 'query=histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, region, verb))'
```

Inspect which label names your deployment actually uses via `/api/v1/labels` first — the conventions above are common but not universal.

## Thanos-specific query params

These extend the standard Prometheus query API:

| Param | Default | Use |
|---|---|---|
| `dedup` | `true` | Dedup HA Prometheus pair series via `replica` label |
| `partial_response` | depends on flag | Return what's available even if some store is down (useful for cross-region availability) |
| `max_source_resolution` | `0s` | Use downsampled blocks: `0s` raw, `5m`, `1h`, or `auto` |
| `replicaLabels` | from CLI | Override the replica label for this query |
| `storeMatch[]` | none | Restrict to specific stores: `{__address__=~"prom-foo.*"}` |
| `engine` | `thanos` | PromQL engine: `prometheus` or `thanos` |

Example — long range query against downsampled data:

```bash
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=avg_over_time(up[30d])' \
  --data-urlencode 'max_source_resolution=1h' \
  "$THANOS_METRICS_URL/api/v1/query" | jq .
```

## Common patterns

```bash
# Service health across all regions
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=count(up == 0) by (region, job)' \
  "$THANOS_METRICS_URL/api/v1/query" | jq '.data.result'

# Error rate across regions (assumes standard label `code`)
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=sum(rate(http_requests_total{code=~"5.."}[5m])) by (region, job)' \
  "$THANOS_METRICS_URL/api/v1/query" | jq '.data.result'

# P99 latency (histogram) — last 5 minutes per region
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, region))' \
  "$THANOS_METRICS_URL/api/v1/query" | jq '.data.result'
```

## Troubleshooting: empty query results

When an instant query or `rate()` query returns `[]`, **do NOT** iteratively widen the time window in small steps (5m → 10m → 30m → 1h). That wastes time and obscures the real cause. Follow this checklist instead:

```
Empty result triage:
- [ ] Step 1: Check stores topology — is there a sidecar for that region?
- [ ] Step 2: Verify the metric name exists at all
- [ ] Step 3: Verify the label value (label names vary across deployments)
- [ ] Step 4: Run a wide range query (6h+) to find the last data point
- [ ] Step 5: Only narrow down once you have an upper bound on the gap
```

### Step 1 — Stores topology first (architecture matters)

Before assuming the query is wrong, check what backs each region:

```bash
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/stores" \
  | jq '.data | to_entries[] | .key as $k | .value[] | {kind:$k, name, lastError, labelSets}'
```

Interpret the result:

| What you see for that region | What it means |
|---|---|
| Has a `sidecar` entry, `lastError: null` | Real-time data available — empty result is a query problem |
| Has a `sidecar` entry, `lastError` populated | Sidecar is broken — that's your answer |
| Only `store` (store-gateway), no `sidecar` | **Historical data only.** Expect ~2h upload delay. `rate()` over last 5m will almost always be empty. |
| Neither sidecar nor store for that region | Region not wired into this Thanos — query the right Querier |

**This single check resolves most "empty result" mysteries.** Do it before anything else.

### Step 2 — Does the metric exist?

```bash
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'match[]={__name__=~".*duration.*bucket"}' \
  "$THANOS_METRICS_URL/api/v1/series?limit=5" \
  | jq '[.data[] | .__name__] | unique'
```

If the metric name is wrong (e.g. `http_request_duration_seconds_bucket` vs `api_request_duration_milliseconds_bucket`), nothing else matters. Confirm the actual name first.

### Step 3 — Do the label values match?

Don't assume `environment="stg"` is the right scope. Inspect actual labels on the metric:

```bash
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'match[]={__name__="<metric_name>"}' \
  "$THANOS_METRICS_URL/api/v1/series?limit=1" | jq '.data[0]'
```

Look for `region`, `thanos_region`, `cluster`, `environment` — different deployments scope differently.

### Step 4 — Wide range query to find the gap

Once topology and naming are confirmed, find the **last data point** with one wide query, not many narrow ones:

```bash
# 6 hours is the default — go wider if needed. NEVER start with 30 minutes.
NOW=$(date -u +%s)
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=count(<metric>{<labels>})' \
  --data-urlencode "start=$(( NOW - 21600 ))" \
  --data-urlencode "end=$NOW" \
  --data-urlencode 'step=1m' \
  "$THANOS_METRICS_URL/api/v1/query_range" \
  | jq '.data.result[0].values | .[-5:] | map({time: (.[0] | todate), count: .[1]})'
```

If still empty, double the window (12h, 24h, 3d). Cheap operation — go wide and bisect, don't crawl forward.

### Step 5 — Now narrow down

Only after you know roughly when data stopped, query the surrounding window with `step=1m` to pinpoint the exact cutoff.

### Anti-patterns

- ❌ Repeatedly retrying `rate(...[5m])` with no data — `rate()` needs ≥2 samples in the window; if scrape stopped, no time window will help.
- ❌ Iteratively expanding `[5m] → [10m] → [30m]` hoping data appears.
- ❌ Querying `/api/v1/query` (instant) after `/api/v1/query_range` showed data — staleness markers may hide instant results even when range queries succeed.
- ❌ Assuming the metric/label is wrong before checking `/api/v1/stores`.

## Notes

- POST endpoints accept `application/x-www-form-urlencoded`; use `--data-urlencode` for queries with special chars.
- `match[]` requires the `[]` suffix.
- Label values endpoint takes label name as path parameter: `/api/v1/label/{name}/values`.
- Response shape is identical to Prometheus HTTP v1 API for `/query`, `/query_range`, `/labels`, `/series`, `/alerts`, `/rules`. `/stores` is Thanos-specific.
- Thanos has **no built-in authentication**. If your deployment is behind a reverse proxy, set `THANOS_AUTH_HEADER` accordingly.
- See [REFERENCE.md](REFERENCE.md) for full endpoint reference and parameters.
- See [EXAMPLES.md](EXAMPLES.md) for end-to-end query examples.
