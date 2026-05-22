# Thanos Query — Examples

End-to-end query examples covering common observability use cases. All examples assume `$THANOS_METRICS_URL` is set in `~/.claude/settings.json`.

> **Label name conventions in these examples**: `region`, `cluster`, `environment`, `job`, `verb`, `code`. Your deployment may use different names — discover them via `/api/v1/labels`.

## 1. Basic connectivity check

```bash
# Health
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/-/healthy"
# expected: OK

# Connected stores summary
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/stores" \
  | jq '.data | to_entries[] | {kind: .key, count: (.value | length)}'

# Unhealthy stores only
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/stores" \
  | jq '.data | to_entries[] as $kind | $kind.value[] | select(.lastError != null) | {kind: $kind.key, name, lastError}'
```

## 2. Discover what data is available

```bash
# All label names
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/labels" | jq -r '.data[]'

# What regions are visible
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/label/region/values" | jq -r '.data[]'

# What metric names match a pattern
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'match[]={__name__=~"http_request.*"}' \
  "$THANOS_METRICS_URL/api/v1/series?limit=20" \
  | jq -r '.data[].__name__' | sort -u
```

## 3. Service health across regions

```bash
# Which (region, job) combinations have any down instances right now
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=count(up == 0) by (region, job) > 0' \
  "$THANOS_METRICS_URL/api/v1/query" \
  | jq '.data.result[] | {region: .metric.region, job: .metric.job, down_count: .value[1]}'

# Service availability ratio per region (last 5m)
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=avg(avg_over_time(up[5m])) by (region)' \
  "$THANOS_METRICS_URL/api/v1/query" \
  | jq '.data.result[] | {region: .metric.region, availability: (.value[1] | tonumber)}'
```

## 4. Latency analysis (histograms)

Assumes histogram bucket metric `http_request_duration_seconds_bucket` with label `verb`.

```bash
# P95 latency per HTTP verb, last 5 minutes
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, verb))' \
  "$THANOS_METRICS_URL/api/v1/query" \
  | jq -r '.data.result[] | "\(.metric.verb)\t\(.value[1])s"'

# P50 / P95 / P99 in one query (returns 3 series per verb)
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, verb)) or histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, verb)) or histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, verb))' \
  "$THANOS_METRICS_URL/api/v1/query" | jq .

# Cross-region P95 — compare regions side by side
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, region))' \
  "$THANOS_METRICS_URL/api/v1/query" \
  | jq -r '.data.result[] | "\(.metric.region)\t\(.value[1])s"'
```

## 5. Error rate

```bash
# Error rate (5xx) per region per service, last 5 minutes
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=sum(rate(http_requests_total{code=~"5.."}[5m])) by (region, job)' \
  "$THANOS_METRICS_URL/api/v1/query" \
  | jq '.data.result[] | {region: .metric.region, job: .metric.job, errors_per_sec: (.value[1] | tonumber)}'

# Error rate as a percentage of total
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=sum(rate(http_requests_total{code=~"5.."}[5m])) by (region) / sum(rate(http_requests_total[5m])) by (region)' \
  "$THANOS_METRICS_URL/api/v1/query" \
  | jq '.data.result[] | {region: .metric.region, error_pct: ((.value[1] | tonumber) * 100)}'
```

## 6. Range query — trend over time

```bash
START=$(date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')
END=$(date -u '+%Y-%m-%dT%H:%M:%SZ')

# Last hour P95 over time, 1 minute step
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))' \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode 'step=1m' \
  "$THANOS_METRICS_URL/api/v1/query_range" \
  | jq '.data.result[0].values | map({ts: (.[0] | strftime("%H:%M:%S")), p95: .[1]})'
```

## 7. Long-range query with downsampling

Useful for monthly dashboards. `max_source_resolution=1h` skips raw and 5m blocks, reading only hourly downsampled blocks — much faster, lower load.

```bash
START=$(date -u -v-30d '+%Y-%m-%dT00:00:00Z' 2>/dev/null || date -u -d '30 days ago' '+%Y-%m-%dT00:00:00Z')
END=$(date -u '+%Y-%m-%dT00:00:00Z')

curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=avg_over_time(up[1d])' \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode 'step=1h' \
  --data-urlencode 'max_source_resolution=1h' \
  "$THANOS_METRICS_URL/api/v1/query_range" | jq '.data.result | length'
```

## 8. Cardinality / cost analysis

```bash
# Top 20 metric names by series count (Thanos aggregates across all stores)
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/status/tsdb" \
  | jq '.data.headStats // .data | (.seriesCountByMetricName // [])[:20]'

# Top label values by cardinality
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/status/tsdb" \
  | jq '.data | (.seriesCountByLabelValuePair // [])[:20]'
```

## 9. Alerts and rules

```bash
# Currently firing alerts grouped by severity
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/alerts" \
  | jq '.data.alerts | group_by(.labels.severity) | map({severity: .[0].labels.severity, count: length})'

# Rule groups with evaluation errors
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/rules" \
  | jq '.data.groups[] | .rules[] | select(.lastError != "" and .lastError != null) | {name, lastError}'
```

## 10. Diagnose "why is this query slow"

```bash
# 1. Check if partial_response is needed (any store down?)
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/api/v1/stores" \
  | jq '.data | to_entries[] as $k | $k.value[] | select(.lastError != null) | .name'

# 2. Try query with explicit timeout + dedup off (faster, less correct)
curl -s ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  --data-urlencode 'query=<your query>' \
  --data-urlencode 'dedup=false' \
  --data-urlencode 'timeout=10s' \
  "$THANOS_METRICS_URL/api/v1/query"

# 3. For long ranges, switch to downsampled
# Add: --data-urlencode 'max_source_resolution=1h'

# 4. Restrict to a specific store
# Add: --data-urlencode 'storeMatch[]={__address__=~"sidecar-region-x.*"}'
```

## Tips

- `--data-urlencode` is safer than putting queries in the URL — works for any special chars.
- When a query returns `{"status":"error", "errorType":"execution", "error":"..."}`, check label name typos first. PromQL silently returns empty for unknown labels in matchers but errors on syntax mistakes.
- `partial_response=true` is a sharp tool — you'll get partial results when a region is down, which may mislead alerting. Default off for alerts, on for dashboards.
- Always `/api/v1/labels` and `/api/v1/stores` first when exploring a new Thanos deployment — discover the schema before guessing.
