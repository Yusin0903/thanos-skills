# thanos-skills

Claude Code skills for [Thanos](https://thanos.io) — query metrics across multiple Prometheus instances with PromQL, directly from the Claude Code CLI.

## What's in here

| Skill | Purpose |
|---|---|
| [`thanos-query`](./skills/thanos-query) | Query Thanos HTTP API: instant/range queries, label/series discovery, stores topology, alerts, rules. |

## Install

Pick whichever method fits your workflow.

### Option 1 — `npx skills` (recommended)

The simplest one-liner, using the [vercel-labs/skills](https://github.com/vercel-labs/skills) installer:

```bash
# Global install (writes to ~/.claude/skills/)
npx skills add Yusin0903/thanos-skills --global

# Or project-local (writes to ./.claude/skills/ in current dir)
npx skills add Yusin0903/thanos-skills
```

`npx skills` auto-discovers skills under `skills/` and installs each one. Re-run the same command to update.

### Option 2 — git clone + symlink

Useful if you want the source on disk to read, modify, or pull updates manually:

```bash
git clone https://github.com/Yusin0903/thanos-skills.git ~/Projects/thanos-skills
ln -s ~/Projects/thanos-skills/skills/thanos-query ~/.claude/skills/thanos-query

# Later, to update:
cd ~/Projects/thanos-skills && git pull
```

### Option 3 — direct copy

No git, no npm — just the files. Useful in restricted environments:

```bash
mkdir -p ~/.claude/skills/thanos-query
curl -L https://github.com/Yusin0903/thanos-skills/archive/refs/heads/main.tar.gz \
  | tar -xz -C /tmp \
  && cp -r /tmp/thanos-skills-main/skills/thanos-query/* ~/.claude/skills/thanos-query/
```

### Verify

```bash
ls ~/.claude/skills/thanos-query/SKILL.md
```

Then start a new Claude Code session — the skill auto-loads from its frontmatter.

## Configure

Skills read your Thanos endpoint from environment variables. Set them in `~/.claude/settings.json` under `env` so every Claude Code session inherits them:

```json
{
  "env": {
    "THANOS_METRICS_URL": "http://thanos.example.internal:10902"
  }
}
```

If your Thanos endpoint is behind a reverse proxy with auth:

```json
{
  "env": {
    "THANOS_METRICS_URL": "https://thanos.example.com",
    "THANOS_AUTH_HEADER": "Authorization: Bearer <your-token>"
  }
}
```

Both variables also work as shell exports if you prefer not to put them in `settings.json`.

## Before you expose Thanos to anything

**Thanos has no built-in authentication.** The HTTP API exposes:

- Read access to every metric across every connected Prometheus / store
- Arbitrary PromQL execution (a single bad query like `count({__name__=~".+"}[30d]) by (__name__)` can OOM the querier or starve store-gateway)
- Internal infrastructure detail in metric labels (instance IPs, pod names, namespaces, service names) — Prometheus's default scrape config leaks all of these

### Required: a network boundary

The "no auth" design is intentional in Thanos — they assume the operator places it behind one of:

1. **Internal-only network** — NLB / ALB with `scheme: internal`, reachable only via VPN, VPC peering, or private connectivity
2. **Reverse proxy with auth** — nginx / oauth2-proxy / vmauth in front, handling auth before traffic reaches Thanos
3. **Service mesh with mTLS** — Istio / Linkerd enforcing peer authentication

**Do not expose Thanos Query to the public internet.** Even on an internal network, treat the HTTP endpoint as "anyone on the network can run any PromQL and read every metric label."

### Recommended hardening (defense in depth)

| Layer | Mitigation |
|---|---|
| Network | `scheme: internal` LB, VPC-scoped Security Group, optional Kubernetes NetworkPolicy |
| Application | Thanos flags: `--query.timeout=2m`, `--query.max-concurrent=20`, `--query.max-samples` |
| Proxy (if added) | Path allowlist (only `/api/v1/query`, `/query_range`, `/labels`, `/series`, `/stores`, `/-/healthy`), rate limit, request size limit |
| Observability | ALB / proxy access logs to S3; Thanos `/metrics` for query rate / latency / error rate |

### Notes for public-by-network deployments

If your Thanos endpoint is reachable from the wider corporate network (any device on VPN), be aware:

- **No audit trail without explicit logging** — Thanos query logs are local to the pod; you'll want a proxy in front (or sidecar) to get per-caller attribution
- **Metric labels may contain sensitive infra detail** — review your scrape configs and `metric_relabel_configs`. Pod names, instance IPs, K8s namespaces are common leakers
- **PromQL is Turing-complete enough to be a DoS vector** — set `--query.timeout` and `--query.max-concurrent` conservatively; consider an outer rate limit if exposed broadly

## Notes for skill users (data flow)

When you run these skills inside Claude Code:

- Skills execute `curl` from your local machine (or wherever Claude Code is running)
- Query responses (including all metric labels) are returned to Claude's context
- That context is processed by Anthropic's API — if your metric labels contain anything you'd consider confidential (internal hostnames, infrastructure CIDRs, customer identifiers via `customer_id` labels, etc.), treat skill output as "information you've shared with Claude"

This is the same trust model as any other Claude Code skill that reads from your environment. Decide whether your Thanos data classification permits it before running broad queries.

## Contributing

Issues and PRs welcome. Each skill should:

- Have a single, focused responsibility (don't add Grafana / Alertmanager flows to `thanos-query`)
- Stay under 100 lines in `SKILL.md` (split into `REFERENCE.md` / `EXAMPLES.md` when needed)
- Use generic label conventions in docs (`region`, `cluster`, `job`) — never hardcode real environment names
- Use the conditional auth pattern (`${THANOS_AUTH_HEADER:+-H} ${THANOS_AUTH_HEADER:+"$THANOS_AUTH_HEADER"}`) so the same skill works for direct-access and proxied deployments

## License

MIT. See [LICENSE](LICENSE).

## See also

- [Thanos documentation](https://thanos.io/tip/components/query.md/)
- [Prometheus HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)
- [Claude Code skills](https://docs.claude.com/en/docs/claude-code)
