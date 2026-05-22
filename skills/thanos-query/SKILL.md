---
name: thanos-query
description: Query Thanos HTTP API for cross-region Prometheus metrics via PromQL. Use when running PromQL/MetricsQL queries against a Thanos Querier or Query Frontend, discovering metrics/labels across regions, inspecting Thanos stores (sidecars/store-gateway), checking alerts/rules, or analyzing time series data that spans multiple Prometheus instances. Triggers on: thanos query, promql, cross-region metrics, p50/p95/p99 latency, sidecar status, store-gateway, query frontend, label discovery across regions.
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

## Notes

- POST endpoints accept `application/x-www-form-urlencoded`; use `--data-urlencode` for queries with special chars.
- `match[]` requires the `[]` suffix.
- Label values endpoint takes label name as path parameter: `/api/v1/label/{name}/values`.
- Response shape is identical to Prometheus HTTP v1 API for `/query`, `/query_range`, `/labels`, `/series`, `/alerts`, `/rules`. `/stores` is Thanos-specific.
- Thanos has **no built-in authentication**. If your deployment is behind a reverse proxy, set `THANOS_AUTH_HEADER` accordingly.
- See [REFERENCE.md](REFERENCE.md) for full endpoint reference and parameters.
- See [EXAMPLES.md](EXAMPLES.md) for end-to-end query examples.
