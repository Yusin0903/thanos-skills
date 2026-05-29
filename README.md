# thanos-skills

![Claude Code](https://img.shields.io/badge/Claude%20Code-skill-8A2BE2)
![Thanos](https://img.shields.io/badge/CNCF-Thanos-6D4AFF)
![PromQL](https://img.shields.io/badge/query-PromQL-E6522C)
![License](https://img.shields.io/badge/license-MIT-3DA639)

**[Claude Code](https://claude.com/claude-code) skills for [CNCF Thanos](https://thanos.io), Prometheus, and PromQL.** Query metrics across multi-region Prometheus through a Thanos Querier, discover labels and stores, debug empty PromQL results, and inspect alerts — without hand-writing every `curl`.

You ask in plain language ("P99 latency for checkout across all regions, last hour"); the skill talks to the Thanos HTTP API, writes the PromQL, and hands the result back to Claude to interpret.

---

## Contents

- [What it does](#what-it-does)
- [What a session looks like](#what-a-session-looks-like)
- [Available skills](#available-skills)
- [Install](#install)
- [Connecting to Thanos](#connecting-to-thanos)
- [Environment variables](#environment-variables)
- [Security](#security)
- [Data flow & privacy](#data-flow--privacy)
- [Repository topics](#repository-topics)
- [License](#license)

---

## What it does

- Query metrics across multiple Prometheus instances through Thanos (instant & range queries).
- Compare latency, error rate, availability, and saturation across regions or clusters.
- Discover label names, label values, metric names, and connected Thanos stores.
- Diagnose empty results caused by missing stores, upload delay, label mismatch, or stale data.
- Inspect active alerts, rules, build/runtime info, flags, and TSDB/cardinality status.

## What a session looks like

```text
You:   What's the P99 latency for the checkout service across all regions in the last hour?

Claude: → checks /api/v1/labels to confirm the metric and `region` label exist
        → runs:
            histogram_quantile(0.99,
              sum by (le, region) (
                rate(http_request_duration_seconds_bucket{service="checkout"}[5m])
              ))
        → returns a per-region breakdown and flags us-west-2 as the outlier (412 ms vs ~180 ms)
```

More prompts that trigger the skill:

- "Show me which stores are connected to Thanos."
- "Discover all label values for `region` in the last 24h."
- "Why does this Thanos query return empty results?"
- "List active alerts from Thanos Ruler."
- "Compare 5xx error rate by Kubernetes cluster."
- "Find whether this metric exists in the store gateway or only in sidecars."

## Available skills

| Skill | Purpose |
|-------|---------|
| [thanos-query](./skills/thanos-query) | Query the Thanos HTTP API: instant/range queries, label & series discovery, stores topology, alerts, rules |

## Install

### Via [skills.sh](https://skills.sh)

```bash
# all skills
npx skills add Yusin0903/thanos-skills

# only the query skill
npx skills add Yusin0903/thanos-skills --skill thanos-query
```

### Via git clone

```bash
git clone https://github.com/Yusin0903/thanos-skills.git ~/Projects/thanos-skills
ln -s ~/Projects/thanos-skills/skills/thanos-query ~/.claude/skills/thanos-query
```

## Connecting to Thanos

The skill runs `curl` from wherever Claude Code is running (your laptop, a dev container, a CI runner). That machine must be able to reach the Thanos Querier HTTP endpoint.

| Deployment | How to reach it |
|------------|-----------------|
| Local dev / docker-compose | `THANOS_METRICS_URL=http://localhost:10902` |
| EKS / GKE internal Service | `kubectl port-forward -n <ns> svc/thanos-query 10902:10902` → `http://localhost:10902` |
| Internal NLB / ALB | Connect to VPN or VPC peering first, then use the LB hostname |
| Behind reverse proxy | Use the public hostname + set `THANOS_AUTH_HEADER` |

### Verify connectivity first

```bash
curl -sS ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/-/healthy"
# expected: OK
```

If this fails, you'll hit one of these when the skill runs:

| Symptom | Likely cause |
|---------|--------------|
| `Could not resolve host` | DNS — VPN not connected, or internal hostname not resolvable from your machine |
| `Connection refused` | Wrong port, or port-forward not running |
| `Connection timed out` | Network path blocked — Security Group, firewall, or no VPC route |
| `401 Unauthorized` / `403 Forbidden` | Reverse proxy in front; `THANOS_AUTH_HEADER` missing or wrong |
| `404 Not Found` on `/-/healthy` | URL points at a non-Thanos endpoint (e.g. Grafana, nginx default page) |
| `SSL certificate verify failed` / `curl: (35)` | Reverse proxy uses a self-signed cert; verify the CA bundle or use a trusted cert |
| HTTP 200 but empty `/api/v1/stores` | Connected to a Querier with no stores wired in — wrong cluster/environment |

## Environment variables

```bash
THANOS_METRICS_URL    # Thanos Querier endpoint (e.g. http://localhost:10902)
THANOS_AUTH_HEADER    # Full HTTP header line, e.g. "Authorization: Bearer <token>"
                      # Leave empty for direct access; set it when behind an auth proxy.
```

## Security

**Thanos has no built-in authentication.** Its HTTP API exposes:

- Read access to every metric across every connected Prometheus / store
- Arbitrary PromQL execution (a single bad query can OOM the Querier)
- Internal infrastructure detail in labels (instance IPs, pod names, namespaces, service names)

Thanos assumes the operator places it behind one of:

1. An internal-only network reachable via VPN or private connectivity
2. A reverse proxy with auth (nginx / oauth2-proxy / vmauth) in front
3. A service mesh with mTLS

**Do not expose Thanos Query to the public internet.**

### General hardening

| Layer | Mitigation |
|-------|------------|
| Application | Thanos flags: `--query.timeout=2m`, `--query.max-concurrent=20`, `--query.max-samples` |
| Proxy | Path allowlist (`/api/v1/query`, `/query_range`, `/labels`, `/series`, `/stores`, `/-/healthy`), rate limit, request size limit |
| Observability | Reverse proxy access logs; Thanos `/metrics` for query rate / latency / error rate |

### AWS-specific hardening

| Layer | Mitigation |
|-------|------------|
| Network | NLB / ALB with `scheme: internal`, VPC-scoped Security Group |
| Kubernetes | NetworkPolicy restricting ingress to known caller pods/namespaces |
| Logging | ALB / proxy access logs shipped to S3 |

## Data flow & privacy

- The skill executes `curl` from wherever Claude Code runs (your machine, container, or CI runner).
- Query responses — **including all metric labels** — are returned to Claude's context.
- That context is processed by Anthropic's API. If your labels contain confidential data (internal hostnames, infrastructure CIDRs, customer identifiers), treat skill output as information shared with Claude.

## Repository topics

Set these on the repo to help others find it:

```text
claude-code  claude-skills  thanos  prometheus  promql
observability  kubernetes  cncf  sre
```

## License

Released under the [MIT License](./LICENSE).
