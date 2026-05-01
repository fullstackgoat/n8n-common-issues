# Self-host reference bundle

> ⏳ **Planned.** Three opinionated `docker-compose.yml` variants + step-by-step README that solves the most common deployment tickets in one place.

## Planned contents

```
selfhost/
├── README.md                     ← 30-step guide covering every gotcha in the Top 10
├── compose.sqlite.yml            ← single-node, SQLite — for evaluation only
├── compose.postgres.yml          ← single-node, Postgres — production-shaped
├── compose.queue-mode.yml        ← main + 2 workers + Redis + Postgres
├── nginx.conf                    ← reverse proxy with WebSocket upgrade headers
├── caddy.Caddyfile               ← Caddy alternative (auto-WebSocket)
├── .env.example                  ← all required env vars, commented
├── scripts/
│   ├── backup.sh                 ← snapshots the .n8n volume + Postgres dump
│   ├── upgrade.sh                ← safe-by-default upgrade procedure
│   └── migrate-sqlite-to-postgres.sh
└── Makefile                      ← `make up` / `make backup` / `make upgrade`
```

## Issue coverage

When complete, this bundle addresses Issues 01, 03, 04, 09 from the [Top 10 Issues report](../docs/top-10-issues.html).

## Status

- [ ] `compose.sqlite.yml` (eval-only with warning banner)
- [ ] `compose.postgres.yml`
- [ ] `compose.queue-mode.yml`
- [ ] nginx + Caddy reference configs (with WebSocket upgrade)
- [ ] backup.sh + upgrade.sh + migrate-sqlite-to-postgres.sh
- [ ] README walkthrough
