# n8n-common-issues

> A field-tested, community-built reference for the most common problems n8n users actually run into — with reproductions, knowledge-base articles, debug tools, and self-hosting blueprints.

[![License: MIT](https://img.shields.io/badge/License-MIT-1c1a17.svg?style=flat-square)](LICENSE)
[![Built for n8n](https://img.shields.io/badge/built%20for-n8n-b14b2c.svg?style=flat-square)](https://n8n.io)
[![Sourced from](https://img.shields.io/badge/sourced%20from-community.n8n.io-6c7a3a.svg?style=flat-square)](https://community.n8n.io)

If you Googled your way here because n8n is misbehaving — start with the [Top 10 Issues report](https://fullstackgoat.github.io/n8n-common-issues/top-10-issues.html). Odds are good your problem is one of them, and there's a fix linked from this repo.

---

## What's inside

| Resource | Where | What it is |
|---|---|---|
| **Top 10 Issues report** | [`docs/top-10-issues.html`](https://fullstackgoat.github.io/n8n-common-issues/top-10-issues.html) | A clustered analysis of ~210 forum threads from `community.n8n.io`, ranked by how often they show up as support tickets. |
| **12-project build plan** | [`docs/projects.html`](https://fullstackgoat.github.io/n8n-common-issues/projects.html) · [`projects.md`](projects.md) | The companion plan: what to build to address every issue, mapped to JD requirements with a coverage matrix. |
| **Reproductions** | [`repros/`](repros/) | One folder per ticket cluster. Each ships a `docker-compose.yml` that boots a clean stack reproducing the issue, plus root-cause and workaround docs. |
| **Knowledge base** | [`kb/`](kb/) | Plain-English fix articles, one per issue. Format: *Symptoms → Confirm → Root cause → Fix → Prevent → Related threads.* |
| **Debug CLI** | [`tools/debug-cli/`](tools/debug-cli/) | Small Node.js CLI for parsing n8n executions, testing webhooks, decoding OAuth tokens, and running network diagnostics. |
| **Self-host bundle** | [`selfhost/`](selfhost/) | Three opinionated `docker-compose.yml` variants (single-node SQLite, single-node Postgres, queue-mode Postgres+Redis), with `WEBHOOK_URL`, healthcheck, backup, and migration scripts done right. |
| **Observability stack** | [`observability/`](observability/) | Pre-wired Prometheus + Loki + Grafana for n8n. Starter dashboard ships with execution latency p50/p95/p99, error rate, queue depth, worker memory. |
| **Guides** | [`guides/`](guides/) | Long-form references for auth debugging, webhook diagnostics, RAG production, and networking fundamentals. |
| **Reports** | [`reports/`](reports/) | Periodic public reports on systemic n8n issues identified from the forum. |

---

## The Top 10 issues

These are the ten clusters this repo is organized around. Click into any folder for the reproduction; click the article link for the plain-English fix.

| # | Issue | Repro | KB article |
|:-:|---|:-:|:-:|
| 01 | Webhook URL & "Connection lost" — the self-hosting onboarding wall | [`repros/01-webhook-url`](repros/01-webhook-url/) | [`kb/01-webhook-url.md`](kb/01-webhook-url.md) |
| 02 | OAuth 2.0 credential failures (Google, Microsoft, custom) | [`repros/02-oauth-callback`](repros/02-oauth-callback/) | [`kb/02-oauth-callback.md`](kb/02-oauth-callback.md) |
| 03 | Docker self-hosting: persistence, permissions, upgrades, migration | [`repros/03-docker-persistence`](repros/03-docker-persistence/) | [`kb/03-docker-persistence.md`](kb/03-docker-persistence.md) |
| 04 | Queue mode & worker scaling regressions | [`repros/04-queue-mode-ghost-triggers`](repros/04-queue-mode-ghost-triggers/) | [`kb/04-queue-mode.md`](kb/04-queue-mode.md) |
| 05 | Webhook delivery, idempotency & the "double trigger" problem | [`repros/05-webhook-double-trigger`](repros/05-webhook-double-trigger/) | [`kb/05-webhook-delivery.md`](kb/05-webhook-delivery.md) |
| 06 | AI Agent / RAG production failures | [`repros/06-rag-vector-dimensions`](repros/06-rag-vector-dimensions/) | [`kb/06-ai-rag.md`](kb/06-ai-rag.md) |
| 07 | Loops, iteration & data-shape errors | [`repros/07-loop-first-item-only`](repros/07-loop-first-item-only/) | [`kb/07-loops.md`](kb/07-loops.md) |
| 08 | HTTP Request & API debugging — proxies, ECONNREFUSED, custom auth | [`repros/08-http-econnrefused`](repros/08-http-econnrefused/) | [`kb/08-http-debug.md`](kb/08-http-debug.md) |
| 09 | Performance, memory & large-data crashes | [`repros/09-memory-overflow-large-sql`](repros/09-memory-overflow-large-sql/) | [`kb/09-performance.md`](kb/09-performance.md) |
| 10 | Provider-specific node bugs & regressions after upgrade | [`repros/10-node-regression-snowflake`](repros/10-node-regression-snowflake/) | [`kb/10-node-regressions.md`](kb/10-node-regressions.md) |

---

## How a reproduction is structured

Every folder in [`repros/`](repros/) follows the same shape so customers, support engineers, and n8n's product team can all use it:

```
repros/01-webhook-url/
├── README.md           ← what the issue looks like to the customer
├── docker-compose.yml  ← boots a clean stack that exhibits the bug
├── REPRO.md            ← exact steps to reproduce, observed vs expected
├── ROOT_CAUSE.md       ← why it happens (with citations)
├── WORKAROUND.md       ← what to do today
└── .env.example        ← all required env vars, commented
```

Run any repro with:

```bash
cd repros/01-webhook-url
cp .env.example .env
docker compose up -d
# follow REPRO.md from here
```

---

## How a KB article is structured

Every article in [`kb/`](kb/) follows the n8n-docs voice and the same template:

```markdown
# [Symptom phrased the way customers Google it]

## Confirm it's this issue
## Why it happens
## Fix
## Prevent it next time
## Related forum threads
```

Designed so a customer can self-serve in under 90 seconds, or a support engineer can paste the link as their reply.

---

## Status

This repo is being built in public. Tracking against the [12-project sprint plan](https://fullstackgoat.github.io/n8n-common-issues/projects.html):

- [x] Top 10 Issues report
- [x] 12-project build plan
- [x] Repository scaffolded
- [ ] Repros 01–10
- [ ] KB articles 01–10
- [ ] `tools/debug-cli` published to npm
- [ ] `selfhost/` reference bundle (3 compose variants)
- [ ] `observability/` starter stack
- [ ] Auth, webhook, RAG, networking guides
- [ ] Q2 2026 systemic issues report

Follow along by starring the repo or watching for releases.

---

## Contributing

Found another issue that should be in here? Open an issue or a PR. The bar for inclusion:

1. **Reproducible** — there must be a `docker-compose.yml` (or equivalent) that boots a stack where the bug shows up
2. **Cited** — link to at least one real `community.n8n.io` thread describing the problem
3. **Actionable** — there's a workaround the customer can apply today, even if no upstream fix exists yet

---

## Methodology

Issue rankings come from a Firecrawl scrape of `community.n8n.io` covering: latest *Questions*, all-time top *Questions*, top *Help me Build my Workflow*, top *Feature Requests*, and the `queue-mode`, `docker`, `webhook`, and `rag` tag pages. Threads are weighted by **count × views × replies**, then filtered through a Senior Support Engineer lens (reproducible, KB-able, customer-facing).

The full methodology and links to every source thread are in [`docs/top-10-issues.html`](https://fullstackgoat.github.io/n8n-common-issues/top-10-issues.html).

---

## License

[MIT](LICENSE) — use anything in here in your own work, your own KB, your own products. Attribution appreciated but not required.

---

## Author

**Paul Gipson** · [paulcertifiedx@gmail.com](mailto:paulcertifiedx@gmail.com) · [@fullstackgoat](https://github.com/fullstackgoat)
