# thanos-skills

Claude Code skills for [Thanos](https://thanos.io) — query metrics across multiple Prometheus instances with PromQL.

## Available Skills

| Skill | Purpose |
|-------|---------|
| [thanos-query](./skills/thanos-query) | Query Thanos HTTP API: instant/range queries, label/series discovery, stores topology, alerts, rules |

## Prerequisites

The skill runs `curl` from wherever Claude Code is running (your laptop, a dev container, a CI runner). That machine must be able to reach the Thanos Querier HTTP endpoint.

Pick the connectivity path that matches your deployment:

| Deployment | How to reach it |
|------------|-----------------|
| Local dev / docker-compose | `THANOS_METRICS_URL=http://localhost:10902` |
| EKS / GKE internal Service | `kubectl port-forward -n <ns> svc/thanos-query 10902:10902` → `http://localhost:10902` |
| Internal NLB / ALB | Connect to VPN or VPC peering first, then use the LB hostname |
| Behind reverse proxy | Use the public hostname + set `THANOS_AUTH_HEADER` |

### Verify before using the skill

```bash
curl -sS ${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"} \
  "$THANOS_METRICS_URL/-/healthy"
# expected: OK
```

If this fails, you will hit one of these failure modes when the skill runs:

| Symptom | Likely cause |
|---------|--------------|
| `Could not resolve host` | DNS — VPN not connected, or internal hostname not resolvable from your machine |
| `Connection refused` | Wrong port, or port-forward not running |
| `Connection timed out` | Network path blocked — Security Group, firewall, or no VPC route |
| `401 Unauthorized` / `403 Forbidden` | Reverse proxy in front; `THANOS_AUTH_HEADER` missing or wrong |
| `404 Not Found` on `/-/healthy` | URL points at a non-Thanos endpoint (e.g. Grafana, nginx default page) |
| `SSL certificate verify failed` / `curl: (35)` | Reverse proxy uses self-signed cert; verify CA bundle or use a trusted cert |
| HTTP 200 but empty `/api/v1/stores` | Connected to Thanos Querier with no stores wired in — wrong cluster/environment |

## Installation

### Via [skills.sh](https://skills.sh)

```
npx skills add Yusin0903/thanos-skills
```

Install a specific skill:

```
npx skills add Yusin0903/thanos-skills --skill thanos-query
```

### Via git clone

```
git clone https://github.com/Yusin0903/thanos-skills.git ~/Projects/thanos-skills
ln -s ~/Projects/thanos-skills/skills/thanos-query ~/.claude/skills/thanos-query
```

## Usage

**Example prompts that trigger the skill:**

- "What's the P99 latency for service X across all regions?"
- "Show me which stores are connected to Thanos"
- "Discover all label values for `region` in the last 24h"
- "Why does this query return empty results?"
- "List active alerts from Thanos Ruler"

## Environment Variables

The skill uses `curl` and expects these environment variables:

```bash
THANOS_METRICS_URL    # Thanos Querier endpoint (e.g., http://localhost:10902)
THANOS_AUTH_HEADER    # Full HTTP header line, e.g. "Authorization: Bearer <token>"
                      # (empty for direct access, set when behind an auth proxy)
```

## Security

**Thanos has no built-in authentication.** The HTTP API exposes:

- Read access to every metric across every connected Prometheus / store
- Arbitrary PromQL execution (a single bad query can OOM the querier)
- Internal infrastructure detail in metric labels (instance IPs, pod names, namespaces, service names)

Thanos assumes the operator places it behind one of:

1. Internal-only network reachable via VPN or private connectivity
2. Reverse proxy with auth (nginx / oauth2-proxy / vmauth) in front
3. Service mesh with mTLS

Do not expose Thanos Query to the public internet.

### General hardening

| Layer | Mitigation |
|-------|------------|
| Application | Thanos flags: `--query.timeout=2m`, `--query.max-concurrent=20`, `--query.max-samples` |
| Proxy | Path allowlist (only `/api/v1/query`, `/query_range`, `/labels`, `/series`, `/stores`, `/-/healthy`), rate limit, request size limit |
| Observability | Reverse proxy access logs; Thanos `/metrics` for query rate / latency / error rate |

### AWS-specific hardening

| Layer | Mitigation |
|-------|------------|
| Network | NLB / ALB with `scheme: internal`, VPC-scoped Security Group |
| Kubernetes | NetworkPolicy restricting ingress to known caller pods/namespaces |
| Logging | ALB / proxy access logs shipped to S3 |

## Data flow

- Skills execute `curl` from your local machine (or wherever Claude Code is running)
- Query responses (including all metric labels) are returned to Claude's context
- That context is processed by Anthropic's API — if your labels contain confidential data (internal hostnames, infrastructure CIDRs, customer identifiers), treat skill output as information shared with Claude

