# Reproductions

One folder per top-10 issue cluster. Each ships a Docker-Compose-driven minimum reproduction with root-cause and workaround docs.

## How to run any of them

```bash
cd repros/<issue-folder>
cp .env.example .env
docker compose up -d
# follow REPRO.md
```

## Status

| # | Issue | Status |
|:-:|---|:-:|
| 01 | [Webhook URL & "Connection lost"](01-webhook-url/) | ✅ done |
| 02 | [OAuth 2.0 callback failures](02-oauth-callback/) | ⏳ planned |
| 03 | [Docker self-hosting persistence](03-docker-persistence/) | ⏳ planned |
| 04 | [Queue mode ghost triggers](04-queue-mode-ghost-triggers/) | ⏳ planned |
| 05 | [Webhook double-trigger / idempotency](05-webhook-double-trigger/) | ⏳ planned |
| 06 | [RAG vector dimension mismatch](06-rag-vector-dimensions/) | ⏳ planned |
| 07 | [Loop only processes first item](07-loop-first-item-only/) | ✅ done |
| 08 | [HTTP Request ECONNREFUSED](08-http-econnrefused/) | ⏳ planned |
| 09 | [Memory overflow on large SQL](09-memory-overflow-large-sql/) | ⏳ planned |
| 10 | [Snowflake Insert regression](10-node-regression-snowflake/) | ⏳ planned |

## Pattern

Every reproduction folder has the same shape — see [`01-webhook-url/`](01-webhook-url/) as the canonical example:

```
NN-issue-name/
├── README.md              ← what the issue looks like to the customer
├── docker-compose.yml     ← boots a clean stack that exhibits the bug
├── docker-compose.fixed.yml  ← (when applicable) the same stack with the fix
├── .env.example           ← all required env vars, commented
├── REPRO.md               ← exact steps, observed vs expected output
├── ROOT_CAUSE.md          ← why it happens (with citations)
└── WORKAROUND.md          ← copy-pasteable fix for a customer instance
```
