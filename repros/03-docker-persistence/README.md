# Repro 03 — Docker self-hosting: persistence, permissions, healthchecks

The most expensive n8n bug, measured in lost work. Three independent failure modes that all look the same to the user — *"my n8n is gone"* — and all stem from one missing block in `docker-compose.yml`.

## Symptoms (any combination)

- Credentials and workflows gone after a routine `docker compose down && up`
- *"Could not decrypt credentials"* error after restoring a backup or moving to a new host
- *"The file 'x' is not writable"* errors from workflows that try to read or write files
- `docker compose ps` says the service is "Up" but the editor is unreachable
- `docker compose down` is treated as a "soft restart" and silently destroys state

## What this repro proves

That all of the symptoms above stem from three things missing from `docker-compose.yml` — a named volume, a pinned `N8N_ENCRYPTION_KEY`, and a `healthcheck` block — plus a UID/GID gotcha that bites bind-mount users specifically. Adding all four fixes every symptom at once.

## Files

| File | Purpose |
|---|---|
| [`docker-compose.yml`](docker-compose.yml) | Broken stack — no volume, no encryption key, no healthcheck. Reproduces data loss in two commands. |
| [`docker-compose.fixed.yml`](docker-compose.fixed.yml) | Fixed stack — named volume, pinned key, healthcheck, explicit `user:`. |
| [`.env.example`](.env.example) | All required env vars, commented. Includes `N8N_ENCRYPTION_KEY` placeholder. |
| [`REPRO.md`](REPRO.md) | Three deterministic symptoms, step-by-step, with observed vs expected output. |
| [`ROOT_CAUSE.md`](ROOT_CAUSE.md) | Why each symptom happens — Docker writable layer, AES key drift, UID/GID on bind-mounts. |
| [`WORKAROUND.md`](WORKAROUND.md) | Copy-pasteable production stack + backup script + full SQLite → Postgres migration. |

## Quick start

```bash
cd repros/03-docker-persistence
cp .env.example .env

# Generate and pin a real encryption key (the fixed stack uses it).
KEY=$(openssl rand -hex 32) && sed -i.bak "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=$KEY|" .env && rm .env.bak

# 1) Boot the broken stack and lose data deliberately
docker compose up -d
# … create owner account + a credential at http://localhost:5678 …
docker compose down && docker compose up -d
# Reload the editor → "create owner account" screen. Data gone.

# 2) Boot the fixed stack and verify persistence survives
docker compose down
docker compose -f docker-compose.fixed.yml up -d
# … create owner account + a credential …
docker compose -f docker-compose.fixed.yml down
docker compose -f docker-compose.fixed.yml up -d
# Reload the editor → owner account and credential are still there.
```

Full step-by-step is in [`REPRO.md`](REPRO.md).

## Linked KB article

[`kb/03-docker-persistence.md`](../../kb/03-docker-persistence.md) — *(planned — Project 2)*. Use [`WORKAROUND.md`](WORKAROUND.md) as the customer-facing fix in the meantime.

## Source threads

- [Credentials lost after Docker restart](https://community.n8n.io/t/credentials-lost-after-docker-restart/226303) — symptom 1, the canonical complaint
- [How to upgrade n8n with docker-compose](https://community.n8n.io/t/how-to-upgrade-n8n-with-docker-compose-setup/660) — 25k views — destructive upgrade procedures
- [How to migrate from SQLite to PostgreSQL](https://community.n8n.io/t/how-to-migrate-from-sqlite-to-postgresql/2170) — 15.4k views — the production upgrade path
- [Read/Write Files from Disk: The file "x" is not writable](https://community.n8n.io/t/read-write-files-from-disk-the-file-x-is-not-writable/267765) — 1.2k views — UID/GID `EACCES`
- [SQLite crash — paid recovery wanted](https://community.n8n.io/t/urgent-docker-hosted-n8n-on-sqlite-crashed-possible-db-corruption-looking-for-paid-expert-to-recover-data-and-migrate-to-postgres/257163) — what unaudited persistence looks like at the failure point
- [Docker image suggestion: add healthcheck](https://community.n8n.io/t/docker-image-suggestion-add-healthcheck/31837) — 3.1k views — community-requested fix still not in the upstream image
