# Build Plan — Project 1: `n8n-repro-lab`

> Working document. Tracks progress on building out the 10 reproductions inside this monorepo.
> Companion to [`projects.md`](projects.md) (the spec) and [`repros/01-webhook-url/`](repros/01-webhook-url/) (the canonical example).
> Internal-facing — not a customer artifact. Live notes; check items off as you go.

---

## Status snapshot

| # | Repro | State | Notes |
|:-:|---|:-:|---|
| 01 | [Webhook URL & "Connection lost"](repros/01-webhook-url/) | ✅ done | Full reference. Pattern-setter for the rest. |
| 02 | [OAuth callback failures](repros/02-oauth-callback/) | ⏳ todo | Wave 2 |
| 03 | [Docker persistence](repros/03-docker-persistence/) | ✅ done | Wave 1 — broken vs fixed compose stacks + full SQLite → Postgres migration in WORKAROUND. |
| 04 | [Queue mode ghost triggers](repros/04-queue-mode-ghost-triggers/) | ⏳ todo | Wave 3 — hardest |
| 05 | [Webhook double trigger](repros/05-webhook-double-trigger/) | ⏳ todo | Wave 2 |
| 06 | [RAG vector dimensions](repros/06-rag-vector-dimensions/) | ⏳ todo | Wave 4 |
| 07 | [Loop first item only](repros/07-loop-first-item-only/) | ✅ done | Wave 1 — second full reference. Three workflow JSONs + full docs + KB. |
| 08 | [HTTP ECONNREFUSED](repros/08-http-econnrefused/) | ⏳ todo | Wave 1 |
| 09 | [Memory overflow](repros/09-memory-overflow-large-sql/) | ⏳ todo | Wave 3 |
| 10 | [Node regression — Snowflake](repros/10-node-regression-snowflake/) | ⏳ todo | Wave 4 |

**Done count:** 3 / 10
**Target completion:** Day 7 of the 14-day sprint
**Cumulative effort burned:** ~5h on Repro 01 + ~2h on Repro 07 + ~3h on Repro 03 = ~10h

---

## What "done" looks like (acceptance criteria)

Every repro must satisfy this checklist before being marked complete. Use it as a self-review before each commit.

```
[ ] README.md                         — symptoms, files, quickstart, source threads
[ ] docker-compose.yml                — broken stack that boots and exhibits the bug
[ ] docker-compose.fixed.yml          — same stack with the fix (or clear note if fix is env-only)
[ ] .env.example                      — every required env var, commented
[ ] REPRO.md                          — numbered steps, observed vs expected for each symptom
[ ] ROOT_CAUSE.md                     — 3-layer explanation (what / why / where it bites)
[ ] WORKAROUND.md                     — copy-pasteable for a ticket reply, per-platform notes
[ ] Smoke-test on clean machine       — `docker compose up -d` boots green from a fresh clone
[ ] Cross-linked from kb/NN-*.md      — KB article exists and references this folder
[ ] Source threads cited              — minimum 3 real community.n8n.io links
[ ] Teardown documented               — `docker compose down -v` cleans up entirely
```

If you can't meet all 11 boxes, the repro is **not done**. Acceptable exception: when the bug requires paid infrastructure (Snowflake, AWS Bedrock), the "stack" can be a mock or documented manual setup — but the rest of the artifacts must still ship.

---

## Build order — 4 waves, 7 days

Sequenced for momentum (easy wins first), then escalating to the hard production-complexity repros that double as the IC3 differentiators.

### Wave 1 — Foundation (Days 1–2): 03, 07, 08

Three easy wins. Knock these out first to build a rhythm and prove the template scales.

- **Repro 07 — Loop first item only** (~2h). Pure workflow JSON; no networking, no auth.
- **Repro 03 — Docker persistence** (~3h). Volume-mount demo with broken vs fixed compose files.
- **Repro 08 — HTTP ECONNREFUSED** (~3h). n8n + fake corporate proxy; demos `host.docker.internal`, proxy env vars, and self-signed certs.

### Wave 2 — Core JD coverage (Days 3–4): 02, 05

Both directly cover JD core requirements (auth + webhooks). Highest interview-relevance.

- **Repro 02 — OAuth callback failures** (~4h). n8n + a fake OAuth IdP container (e.g. `mock-oauth2-server`).
- **Repro 05 — Webhook double trigger** (~3h). n8n + a fake provider container that retries on slow responses.

### Wave 3 — Production complexity (Days 5–6): 04, 09

The IC3 differentiators. Hardest to reproduce deterministically. Worth the effort because they signal "I can debug production systems."

- **Repro 04 — Queue mode ghost triggers** (~5h). main + 2 workers + Redis + Postgres. Tricky.
- **Repro 09 — Memory overflow** (~3h). Seeded Postgres + workflow that loads all rows at once vs. streams.

### Wave 4 — AI & close-out (Day 7): 06, 10

- **Repro 06 — RAG vector dimensions** (~4h). n8n + Qdrant + a known-good corpus + RAGAS-style eval.
- **Repro 10 — Snowflake regression** (~2h). Documentation + workflow JSON; no live Snowflake.

**Total budgeted effort:** ~29h over 7 days ≈ 4h/day. Realistic with a part-time schedule.

---

## Per-repro build briefs

Each brief tells you exactly what to build. Open the matching folder and follow the brief; don't reinvent the structure.

### 🟢 Wave 1

#### Repro 03 — Docker persistence

**Goal proven:** Credentials survive restart only when (a) named volume is mounted, (b) encryption key is persisted, (c) UID/GID match.

**Stack:** n8n (single node) + a host-bind-mount with wrong UID to demonstrate the file-not-writable error.

**Two compose files:**

- `docker-compose.yml` — broken: bind-mounts to `/tmp/n8n-bad` with no UID hint; no `N8N_ENCRYPTION_KEY` set; no healthcheck.
- `docker-compose.fixed.yml` — fixed: named volume `n8n_data:`; `N8N_ENCRYPTION_KEY` from env; `user: "1000:1000"`; healthcheck added.

**REPRO.md walks through:**

1. Boot broken stack, create a credential.
2. `docker compose down && up` — credential is gone (no volume) or unreadable (encryption key changed).
3. Try to write a file from a workflow — `EACCES` because UID inside container ≠ host bind-mount owner.
4. Switch to fixed stack → all symptoms gone.

**Specific files in this folder:**
```
03-docker-persistence/
├── README.md
├── docker-compose.yml
├── docker-compose.fixed.yml
├── .env.example       # N8N_ENCRYPTION_KEY=<openssl rand -hex 32>
├── REPRO.md
├── ROOT_CAUSE.md
└── WORKAROUND.md
```

**Done criteria specific to this repro:**
- [ ] Both compose stacks boot on a clean machine
- [ ] The credential-loss step is reproducible 100% of the time
- [ ] WORKAROUND.md includes the SQLite → Postgres migration script (a one-liner using `pgloader` or n8n's CLI export)

---

#### Repro 07 — Loop first item only

**Goal proven:** "Loop Over Items" is iterating correctly; the user's mental model is wrong about how items flow downstream.

**Stack:** n8n (single node) only. No external dependencies.

**This is the simplest repro.** Most of the value is in the *workflow JSON* and the explanation, not the stack.

**Files:**
```
07-loop-first-item-only/
├── README.md
├── docker-compose.yml         # vanilla n8n, just for running the workflows
├── .env.example
├── workflows/
│   ├── broken-loop.json       # User's mental model: Loop wraps the inner work
│   ├── fixed-loop.json        # Correct: every node downstream of the trigger already loops per item
│   └── three-keys-error.json  # Reproduces "Input values have 3 keys" with Set + sub-workflow
├── REPRO.md
├── ROOT_CAUSE.md              # Explain n8n's items model, "fan-out" semantics
└── WORKAROUND.md
```

**REPRO.md walks through:**
1. Import `broken-loop.json`. Run with 5 input items. Note that only the first downstream effect happens.
2. Show that adding "Loop Over Items" before the work changes nothing — it was already looping.
3. Import `fixed-loop.json`. Show all 5 effects.
4. Import `three-keys-error.json`. Show the exact error, then explain.

**Done criteria specific to this repro:**
- [ ] All 3 workflow JSONs export correctly via n8n's UI export
- [ ] ROOT_CAUSE.md explains the items model in <500 words with a diagram (ASCII or Mermaid)
- [ ] Cite the [Loop Over Items improvement thread (n8n staff)](https://community.n8n.io/t/have-your-say-help-us-improve-the-loop-over-items-node/92570)

---

#### Repro 08 — HTTP ECONNREFUSED

**Goal proven:** ECONNREFUSED is almost always (a) container can't reach `localhost`, (b) proxy not set, or (c) self-signed cert.

**Stack:** n8n + a fake "corporate proxy" container (Squid) + a fake "internal API" that's only reachable through the proxy.

**Three sub-scenarios** (all in one compose stack, distinguished by which workflow you import):

1. **Ollama-style:** workflow tries `http://localhost:11434` from inside n8n container → fails. Fix: `http://host.docker.internal:11434`.
2. **Corporate proxy:** workflow tries to reach `https://internal.corp.example` → fails until `HTTP_PROXY` / `HTTPS_PROXY` are set on the n8n container.
3. **Self-signed cert:** workflow hits `https://self-signed.example` → fails. Fix: mount CA bundle or set `NODE_TLS_REJECT_UNAUTHORIZED=0` (with a giant warning about prod).

**Files:**
```
08-http-econnrefused/
├── README.md
├── docker-compose.yml
├── docker-compose.fixed.yml
├── .env.example
├── workflows/
│   ├── 1-ollama-localhost.json
│   ├── 2-corp-proxy.json
│   └── 3-self-signed.json
├── REPRO.md
├── ROOT_CAUSE.md
└── WORKAROUND.md
```

**Done criteria specific to this repro:**
- [ ] All three scenarios fail deterministically on `docker-compose.yml`
- [ ] All three scenarios succeed on `docker-compose.fixed.yml`
- [ ] WORKAROUND.md has copy-pasteable env-var blocks for each scenario

---

### 🟡 Wave 2

#### Repro 02 — OAuth callback failures

**Goal proven:** OAuth callback URL must match what's configured at the IdP, and refresh-token expiry is a real failure mode that needs handling.

**Stack:** n8n + a mock OAuth2 server (e.g. [`navikt/mock-oauth2-server`](https://github.com/navikt/mock-oauth2-server) or Keycloak).

**Two scenarios:**

1. **Callback URL mismatch:** n8n boots with default config, OAuth client is registered at IdP with `https://workflows.example.com/...`, n8n emits `localhost:5678/...`. OAuth completes but redirects nowhere useful.
2. **Refresh token expiry:** Mock IdP issues a 60-second token. After 60s, the next workflow run that uses the credential fails with "invalid_token". Show how to detect and how to refresh.

**Files:**
```
02-oauth-callback/
├── README.md
├── docker-compose.yml          # n8n + mock-oauth2-server, n8n misconfigured
├── docker-compose.fixed.yml    # n8n + mock-oauth2-server, n8n correctly configured
├── .env.example
├── mock-oauth2-config.json
├── REPRO.md
├── ROOT_CAUSE.md
└── WORKAROUND.md
```

**Done criteria specific to this repro:**
- [ ] OAuth dance completes end-to-end on the fixed stack
- [ ] Refresh-token expiry is reproducible by waiting 60s
- [ ] WORKAROUND.md includes the `allowedDomains` Public-API gotcha
- [ ] WORKAROUND.md includes ngrok-specific guidance

---

#### Repro 05 — Webhook double trigger

**Goal proven:** Without an idempotency key, providers retrying on slow/failed responses cause workflows to run multiple times and double-effect downstream.

**Stack:** n8n + a tiny custom Node.js container called `flaky-provider` that POSTs to n8n's webhook with realistic provider semantics:

- Retries 3× with exponential backoff if it doesn't get a 2xx within 5s
- Re-sends the same payload (idempotent from provider's POV)
- Optional `--slow` flag to make n8n's webhook take 10s to respond, triggering retry

**Two workflows:**

- `non-idempotent.json` — POSTs to a fake CRM that creates a lead. Run it 3× → 3 leads created.
- `idempotent.json` — Uses Set + `If` + a Postgres dedupe table keyed off the provider's request-id header. Run 3× → 1 lead created.

**Files:**
```
05-webhook-double-trigger/
├── README.md
├── docker-compose.yml
├── flaky-provider/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── workflows/
│   ├── non-idempotent.json
│   └── idempotent.json
├── REPRO.md
├── ROOT_CAUSE.md
└── WORKAROUND.md
```

**Done criteria specific to this repro:**
- [ ] `flaky-provider` boots and reliably triggers retries
- [ ] Non-idempotent workflow demonstrates double-effect 100% of the time
- [ ] Idempotent workflow correctly deduplicates
- [ ] WORKAROUND.md covers all four common idempotency patterns: request-id dedupe, content hash, optimistic upsert, transactional outbox

---

### 🔴 Wave 3

#### Repro 04 — Queue mode ghost triggers

**Hardest repro.** Budget extra time. Race conditions are non-deterministic by definition; you'll need to engineer the timing.

**Goal proven:** When n8n runs with multiple workers, certain triggers (cron especially) can register on every worker instead of only the main process, causing N×duplicate firings on every tick.

**Stack:** n8n main + 2 workers + Redis + Postgres. All in one compose file; toggle `EXECUTIONS_MODE=queue` and worker concurrency.

**Two configurations:**

- `docker-compose.yml` — workers configured to also load triggers (the bug). A cron set to every 30s fires 3 times per tick.
- `docker-compose.fixed.yml` — workers configured with `N8N_DISABLE_PRODUCTION_MAIN_PROCESS=true` only on workers and the env-var that disables active workflows on workers. Cron fires once.

**REPRO.md walks through:**
1. Boot broken stack. Activate a workflow with cron trigger every 30s and a single Postgres "INSERT into events" node.
2. Watch the events table — 3 rows per tick instead of 1.
3. Apply fix → 1 row per tick.

**Files:**
```
04-queue-mode-ghost-triggers/
├── README.md
├── docker-compose.yml          # main + 2 broken workers
├── docker-compose.fixed.yml    # main + 2 correct workers
├── .env.example
├── workflows/
│   └── cron-insert.json
├── postgres-init.sql           # creates the 'events' table
├── REPRO.md
├── ROOT_CAUSE.md               # cite the v2.2.4 race condition thread
└── WORKAROUND.md
```

**Done criteria specific to this repro:**
- [ ] Bug reproduces 100% of the time within ~60s
- [ ] Fix eliminates duplicates verifiably
- [ ] ROOT_CAUSE.md cites the specific n8n version where the race was introduced
- [ ] WORKAROUND.md includes the "isolate workflows to dedicated workers" pattern

---

#### Repro 09 — Memory overflow

**Goal proven:** Workflows that load entire result sets into a single item OOM-kill containers; chunked SELECT + filesystem binary mode survives.

**Stack:** n8n with `mem_limit: 512m` + Postgres seeded with 50,000 rows of synthetic data via `postgres-init.sql`.

**Two workflows:**

- `naive.json` — `SELECT * FROM big_table` → Loop → HTTP request. OOM-killed.
- `streamed.json` — `SELECT * FROM big_table LIMIT 500 OFFSET ${offset}` in a loop → Loop → HTTP request. Survives.

**REPRO.md walks through:**
1. Boot stack. Verify Postgres has 50k rows.
2. Run `naive.json`. Watch `docker stats` — RSS climbs, container OOM-killed within ~30s.
3. Run `streamed.json`. RSS stays flat, completes successfully.

**Files:**
```
09-memory-overflow-large-sql/
├── README.md
├── docker-compose.yml
├── postgres-init.sql           # generates 50k synthetic rows
├── workflows/
│   ├── naive.json
│   └── streamed.json
├── REPRO.md
├── ROOT_CAUSE.md
└── WORKAROUND.md
```

**Done criteria specific to this repro:**
- [ ] OOM kill is reproducible
- [ ] WORKAROUND.md covers: chunked SELECT, binary-data filesystem mode, `Loop Over Items` batch size, `EXECUTIONS_DATA_SAVE_ON_*` settings

---

### 🟣 Wave 4

#### Repro 06 — RAG vector dimensions

**Goal proven:** Most "RAG isn't working" tickets are dimension mismatches, embedding-API failures, or chunking problems — diagnosable with a small eval harness.

**Stack:** n8n + Qdrant + a known-good corpus (~50 markdown docs about cooking) + a tiny Node.js eval container that runs precision@k against a fixed query set.

**Three sub-scenarios:**

1. **Dimension mismatch:** OpenAI embedding model returns 1536, Qdrant collection created with 192 → insert fails or stores garbage. Fix: recreate collection with correct dim.
2. **Embeddings node not firing:** workflow looks correct but no embedding API calls happen on inserts (cite the forum thread).
3. **Bad chunking:** corpus chunked with `chunk_size=100` produces fragmented, contextless retrieval. Fix: `chunk_size=500, overlap=50`.

**Files:**
```
06-rag-vector-dimensions/
├── README.md
├── docker-compose.yml
├── corpus/                     # ~50 .md files about cooking (synthetic / public-domain)
├── eval/
│   ├── Dockerfile
│   ├── eval.js                 # precision@k against fixed Q&A pairs
│   └── qa-pairs.json
├── workflows/
│   ├── 1-dim-mismatch.json
│   ├── 2-no-embedding.json
│   └── 3-bad-chunks.json
├── REPRO.md
├── ROOT_CAUSE.md
└── WORKAROUND.md
```

**Done criteria specific to this repro:**
- [ ] Eval harness produces a green/red score in under 60s
- [ ] All three scenarios reproduce deterministically
- [ ] WORKAROUND.md includes the "RAG production checklist" (links to Project 8 once that exists)

---

#### Repro 10 — Snowflake / provider node regression

**Lightest repro — almost entirely documentation.** Snowflake requires real credentials and you can't ship those.

**What to build instead:**

- A `REGRESSION_LOG.md` documenting the diff between two n8n versions for one specific node bug (Snowflake Insert breaking on 2.17.5 is the canonical example)
- A workflow JSON that demonstrates the breaking change (uses generic provider that anyone can run)
- A `triage-script.sh` that:
  1. Pulls two adjacent n8n image tags
  2. Runs the same workflow against both via n8n's CLI execute mode
  3. Diffs the outputs

**Files:**
```
10-node-regression-snowflake/
├── README.md
├── REGRESSION_LOG.md           # documented regressions across recent n8n versions
├── docker-compose.yml          # vanilla n8n, version pinned via env var
├── workflows/
│   └── reproduces-regression.json
├── triage-script.sh            # runs the workflow against two n8n versions and diffs
├── REPRO.md
├── ROOT_CAUSE.md
└── WORKAROUND.md               # how to roll back, how to file a bug report n8n eng will love
```

**Done criteria specific to this repro:**
- [ ] `triage-script.sh` is general-purpose — works for any future node regression
- [ ] `REGRESSION_LOG.md` documents at least 3 real regressions from the source threads (Snowflake, Salesforce, Gemini 3.0)
- [ ] WORKAROUND.md is essentially "how to file a bug report that gets fixed fast" — links to n8n's GitHub issue template

---

## Reusable templates

Copy-paste these to start each new repro folder.

### `docker-compose.yml` skeleton (broken)

```yaml
name: n8n-repro-NN-<slug>-broken

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "${PUBLIC_PORT:-5678}:5678"
    environment:
      - GENERIC_TIMEZONE=America/New_York
      - N8N_RUNNERS_ENABLED=true
      - N8N_DIAGNOSTICS_ENABLED=false
      # Intentionally minimal — the bug shows up because of what's MISSING here.
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

### `REPRO.md` template

```markdown
# REPRO — Repro NN

## Setup

\`\`\`bash
cd repros/NN-<slug>
cp .env.example .env
docker compose up -d
\`\`\`

## Symptom 1 — <name>

**Observed:** <what happens>

**Expected:** <what should happen>

**Why we see it:** <one-paragraph technical explanation>

## Symptom 2 — <name>
...

## Apply the fix

\`\`\`bash
docker compose down
docker compose -f docker-compose.fixed.yml up -d
\`\`\`

Reload and verify each symptom is gone.

## Teardown

\`\`\`bash
docker compose -f docker-compose.fixed.yml down -v
\`\`\`
```

### Exporting workflows from n8n

When the repro includes a workflow JSON:

1. Build the workflow in the running n8n container
2. In the editor: *Workflow menu → Download → Save as `workflows/<name>.json`*
3. Open the JSON; remove `id`, `versionId`, `meta.instanceId`, and any credential references that would leak local data

For credentials referenced inside workflows, leave them as `"name": "Some Credential"` placeholders. The reader will configure their own.

---

## Common pitfalls (read before starting any repro)

1. **`docker compose` (v2) vs `docker-compose` (v1).** Use `docker compose`. Some Mac users still have v1 aliased; warn in REPRO.md.
2. **Apple Silicon image pulls.** n8n's official image is multi-arch but some upstream deps (Postgres, Redis) sometimes need `platform: linux/amd64` on M-series. Add it preemptively if you see boot loops.
3. **Port collisions.** Don't hardcode `5678` for the published port — use `${PUBLIC_PORT:-5678}` so users with multiple repros booted at once don't collide.
4. **Volume cleanup.** `docker compose down` doesn't remove volumes by default. REPRO.md teardown must say `down -v` or your second run uses stale data.
5. **`host.docker.internal` on Linux.** It works on Docker Desktop (macOS, Windows) but NOT on Linux without `extra_hosts: ["host.docker.internal:host-gateway"]`. Add this preemptively.
6. **Encryption-key drift.** If you don't set `N8N_ENCRYPTION_KEY` explicitly, n8n generates one and stores it in the volume. Across compose-down-and-up, if the volume persists but the env var isn't set, credentials decrypt fine. If the volume is wiped *or* you set a different key, credentials become unreadable.
7. **Workflow imports requiring credentials.** Reader can't run a workflow that depends on a credential they haven't created. Either include a setup step in REPRO.md, or pick scenarios that don't need credentials, or use mock services in the compose stack.

---

## Daily kanban

Check off as you ship. Each row = one work session.

### Day 1
- [x] **AM:** Repro 07 — workflow JSONs, README, ROOT_CAUSE
- [x] **PM:** Repro 07 — REPRO, WORKAROUND, smoke test, commit, push
- [x] **Evening:** Cross-link from `kb/07-loops.md` (stub the KB if needed) — full KB article shipped, not a stub

### Day 2
- [x] **AM:** Repro 03 — both compose files, .env.example
- [x] **PM:** Repro 03 — REPRO, ROOT_CAUSE, WORKAROUND, smoke test
- [ ] **Evening:** Repro 08 scaffolding (just folder + README)

### Day 3
- [ ] **AM:** Repro 08 — three scenarios, compose stacks
- [ ] **PM:** Repro 08 — REPRO, ROOT_CAUSE, WORKAROUND
- [ ] **Evening:** Repro 02 scaffolding + read mock-oauth2-server docs

### Day 4
- [ ] **AM:** Repro 02 — both compose stacks with mock OAuth IdP
- [ ] **PM:** Repro 02 — REPRO, ROOT_CAUSE, WORKAROUND
- [ ] **Evening:** Repro 05 scaffolding + sketch flaky-provider container

### Day 5
- [ ] **AM:** Repro 05 — flaky-provider Dockerfile + server.js
- [ ] **PM:** Repro 05 — workflows, REPRO, ROOT_CAUSE, WORKAROUND
- [ ] **Evening:** Repro 04 scaffolding (compose with main+workers)

### Day 6
- [ ] **AM:** Repro 04 — broken stack + cron workflow + reproduce the bug
- [ ] **PM:** Repro 04 — fixed stack, REPRO, ROOT_CAUSE, WORKAROUND
- [ ] **Evening:** Repro 09 scaffolding (postgres-init.sql with 50k rows)

### Day 7
- [ ] **AM:** Repro 09 — naive vs streamed workflows + REPRO
- [ ] **PM:** Repro 06 — Qdrant compose + corpus + eval harness
- [ ] **Evening:** Repro 10 — REGRESSION_LOG.md + triage-script.sh

### Day 7 close-out checklist
- [ ] All 10 repros pass the 11-point acceptance criteria
- [ ] Root README.md status table updated to all ✅
- [ ] `repros/README.md` status table updated
- [ ] One final smoke pass: clone the repo to a fresh directory, run all 10 repros sequentially
- [ ] Tag a release: `git tag v0.1.0-repros-complete && git push --tags`

---

## Definition of "Project 1 complete"

Project 1 is **shipped** when all of the following are true:

1. Every repro 01–10 has all 11 acceptance-criteria boxes checked
2. The smoke pass on a fresh clone works end-to-end
3. Each repro is cross-linked from a KB stub (the full KB articles are Project 2; just the stub link is enough for Project 1)
4. The root `README.md` status checklist shows the Repros line as ✅
5. A `v0.1.0-repros-complete` git tag exists on `main`

At that point, this file (`build-project1.md`) can be moved to `_archive/` or deleted — its work is done.

---

## Notes & decisions log

> Append free-form notes here as the build progresses. Useful for the systemic-issues report (Project 11) later.

- _2026-05-01:_ Repro 01 (webhook URL) shipped. ~5h. Pattern feels right; copying it for the rest.
- _2026-05-04:_ Repro 07 (loops & "N keys" error) shipped. ~2h. Workflow-only repro — three importable JSONs (broken-loop, fixed-loop, three-keys-error) + full docs + KB article. Pattern from Repro 01 ports cleanly; the workflow-JSON-with-sticky-notes format works well for forum-style bugs that don't need a docker stack to reproduce. Also confirmed the ~2h budget is realistic when the bug is purely in workflow design.
- _2026-05-04:_ Repro 03 (docker persistence) shipped. ~3h. Broken stack reproduces data loss in two commands (no volume + no encryption key + no healthcheck); fixed stack adds named volume, pinned `N8N_ENCRYPTION_KEY`, healthcheck, and explicit `user: "1000:1000"`. WORKAROUND.md covers the full SQLite → Postgres migration with the official `n8n export:credentials/workflow` CLI commands, plus per-platform notes (Linux / macOS / NAS / k8s). KB stub stays planned (Project 2). All three symptoms are deterministic — no flaky timing, no platform-specific gotchas in the core repro path. **Smoke-test passed end-to-end on Docker 29.4.1 / Compose v5.1.3:** broken stack → create owner+credential at `localhost:5678` → `docker compose down && up` → owner-setup screen returns (data gone, confirmed). Fixed stack → create owner+credential → `docker compose down && up` → owner account preserved, credential `repro-credential` still present and decryptable. Bonus evidence: broken stack `docker compose ps` shows `Up Xs` with no health column; fixed stack transitions `(health: starting)` → `(healthy)` at ~35s. Named volume `n8n-repro-03-docker-persistence-fixed_n8n_data` survives `down`, removed by `down -v`. All three symptoms reproduce 100%; all three fixes verified.
- _<date>:_ ...
