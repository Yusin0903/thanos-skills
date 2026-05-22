# Thanos HTTP API — Full reference

This is a working-level reference for the Thanos Querier / Query Frontend HTTP API. Source: <https://thanos.io/tip/components/query.md/> + upstream Prometheus HTTP API.

## Conventions

- Base URL = `$THANOS_METRICS_URL` (set in `~/.claude/settings.json` → `env`).
- All commands use the conditional auth pattern. If your Thanos is behind a reverse proxy, set `THANOS_AUTH_HEADER`; otherwise leave it unset.
- Times accept RFC3339 (`2026-03-07T09:00:00Z`) or unix seconds (`1709769600`).
- POST endpoints accept `application/x-www-form-urlencoded`. Use `--data-urlencode` for special characters.

## Query

### `GET|POST /api/v1/query` — instant query

| Param | Type | Required | Default | Notes |
|---|---|---|---|---|
| `query` | PromQL | yes | — | The PromQL expression |
| `time` | RFC3339 or unix | no | now | Evaluation timestamp |
| `timeout` | duration | no | server flag | Per-query timeout, capped by `--query.timeout` |
| `dedup` | bool | no | `true` | Dedup HA Prometheus replicas via the `replica` label |
| `partial_response` | bool | no | server flag | Return partial data if some stores are unavailable |
| `max_source_resolution` | `0s` / `5m` / `1h` / `auto` | no | `0s` | Choose downsampled resolution for long ranges |
| `replicaLabels` | repeated | no | server flag | Replace the default replica label set |
| `storeMatch[]` | matcher | no | none | Restrict query to matching stores |
| `engine` | `prometheus` or `thanos` | no | server flag | Select PromQL engine |
| `lookback_delta` | duration | no | server flag | Override staleness lookback |

Response (success):
```json
{
  "status": "success",
  "data": {
    "resultType": "vector|matrix|scalar|string",
    "result": [ { "metric": {...}, "value": [ts, "val"] } ]
  },
  "warnings": []
}
```

### `GET|POST /api/v1/query_range` — range query

Same params as `/query` plus:

| Param | Type | Required |
|---|---|---|
| `start` | RFC3339 or unix | yes |
| `end` | RFC3339 or unix | yes |
| `step` | duration (`15s`, `1m`, `5m`, …) | yes |

Response wraps each series with arrays of `values` and `timestamps`.

### `GET /api/v1/format_query`

Format / pretty-print a PromQL query. Returns the canonical form. Useful for normalizing before storage in alert rules.

## Discovery

### `GET /api/v1/labels`

All label names known to the querier.

Optional params: `match[]`, `start`, `end`.

### `GET /api/v1/label/{name}/values`

Values of a single label. Path parameter is the label name. Optional `match[]`, `start`, `end`, `limit`.

### `GET /api/v1/series`

Series matching one or more matchers.

| Param | Type | Required |
|---|---|---|
| `match[]` | matcher | yes (one or more) |
| `start` | RFC3339 or unix | no |
| `end` | RFC3339 or unix | no |
| `limit` | int | no |

### `GET /api/v1/metadata`

Metric metadata (HELP, TYPE). Optional `metric`, `limit`.

## Thanos-specific

### `GET /api/v1/stores`

Lists all stores connected to the Querier, grouped by type (`sidecar`, `store`, `rule`, `receive`, `exemplar`, `metric-metadata`, `target`).

Per-store fields:
- `name` — gRPC endpoint
- `lastCheck` — timestamp of last health probe
- `lastError` — null on healthy, error string on failure
- `labelSets` — external labels exposed by that store
- `minTime`, `maxTime` — time range coverage

Use this to verify a sidecar / store-gateway is connected and which regions / labels it advertises.

### `GET /api/v1/targets`

Scrape target status (only available when the Querier is configured to surface targets from upstreams; sidecar mode forwards from the Prometheus behind it).

### `GET /api/v1/alerts`

Currently firing / pending alerts.

### `GET /api/v1/rules`

Recording and alerting rule groups loaded by Ruler (if connected via `--rule` flag).

## Status

### `GET /api/v1/status/buildinfo`

Version, commit, build platform.

### `GET /api/v1/status/runtimeinfo`

Goroutines, GC stats, uptime.

### `GET /api/v1/status/flags`

Command-line flag values.

### `GET /api/v1/status/tsdb`

Aggregate TSDB stats across visible stores: top series counts by metric name, label name, label value pair. Useful for cardinality analysis.

### `GET /-/healthy` and `GET /-/ready`

Health endpoints. Return 200 OK when the querier is responsive.

## Receive-specific (only if `/receive` is in front)

These endpoints exist on Thanos Receive, not on the Querier. If your `THANOS_METRICS_URL` points at a Querier, they 404.

- `POST /api/v1/receive` — remote_write ingest
- `GET /api/v1/receive/hashring` — current hashring config

## Common response shapes

### Vector result (instant query)
```json
{"metric": {"__name__":"up","job":"api","region":"us-west"}, "value": [1709769600.0, "1"]}
```

### Matrix result (range query)
```json
{"metric": {...}, "values": [[t1,"v1"], [t2,"v2"], ...]}
```

### Error response
```json
{"status":"error","errorType":"bad_data","error":"parse error at char 1: unexpected EOF"}
```

Common `errorType` values: `bad_data`, `execution`, `internal`, `unavailable` (partial_response disabled and a store failed), `timeout`.

## jq cookbook

```bash
# Pull labels off a vector result
... | jq '.data.result[] | {labels: .metric, value: (.value[1] | tonumber)}'

# Group counts by one label
... | jq '.data.result | group_by(.metric.region) | map({region: .[0].metric.region, count: length})'

# Range query — extract series + first/last point
... | jq '.data.result[] | {metric: .metric, first: .values[0], last: .values[-1]}'

# Stores list — flatten across kinds, show unhealthy only
... | jq '.data | to_entries[] as $kind | $kind.value[] | select(.lastError != null) | {kind: $kind.key, name, lastError}'

# Distinct regions seen by querier
curl -s "$THANOS_METRICS_URL/api/v1/label/region/values" | jq -r '.data[]'
```

## Error handling

- HTTP 200 with `status: "error"` — query-level error (bad PromQL, store unavailable). Inspect `errorType` and `error`.
- HTTP 422 — invalid parameters before query runs.
- HTTP 503 — Querier overloaded (`--query.max-concurrent` exceeded) or `partial_response=false` and a store is down.
- HTTP 504 — query exceeded `--query.timeout`.
- HTTP 401 / 403 — auth required by a reverse proxy in front; set `THANOS_AUTH_HEADER`.

## See also

- [SKILL.md](SKILL.md) — top-level instructions
- [EXAMPLES.md](EXAMPLES.md) — practical end-to-end examples
