# Observability starter stack

> ⏳ **Planned.** Pre-wired Prometheus + Loki + Grafana for n8n. Starter dashboard ships with the metrics that matter for triaging support tickets.

## Planned dashboard panels

- Execution latency p50 / p95 / p99 (per workflow)
- Execution error rate (per workflow)
- Queue depth + worker concurrency
- Worker memory / CPU
- Redis ops/sec + connection count
- Postgres connection pool utilization
- Webhook 4xx/5xx rate
- AI Agent token usage + latency

## Files (planned)

```
observability/
├── README.md
├── compose.yml                 ← n8n + Prometheus + Loki + Grafana, all wired
├── prometheus/
│   └── prometheus.yml
├── loki/
│   └── loki-config.yml
├── grafana/
│   ├── provisioning/datasources/
│   ├── provisioning/dashboards/
│   └── dashboards/n8n-support.json    ← the starter dashboard
└── docs/
    └── triage-methodology.md   ← how to use this stack to triage a slow-workflow ticket in <10 min
```

## Issue coverage

When complete, this stack addresses Issues 04 and 09 from the [Top 10 Issues report](../docs/top-10-issues.html).
