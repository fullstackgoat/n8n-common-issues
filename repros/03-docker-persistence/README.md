# Repro 03 — Docker self-hosting: persistence, permissions, upgrades

> ⏳ **Planned.** See [`repros/01-webhook-url/`](../01-webhook-url/) for the pattern this folder will follow.

## What this will reproduce

The Docker persistence/permissions cluster:

- Credentials lost after `docker compose down && up` (encryption-key not persisted)
- File-write permission errors (`/home/node/.n8n/...` not writable)
- SQLite database corruption under concurrent load
- Upgrade procedure that destroys data
- n8n unreachable from another computer (bind address issue)

The reproduction will boot two stacks side-by-side: one with broken volume/permission setup, one with the correct setup (named volume + correct UID/GID + healthcheck + backup hook).

## Source threads

- [How to upgrade n8n with docker-compose](https://community.n8n.io/t/how-to-upgrade-n8n-with-docker-compose-setup/660) — 25k views
- [SQLite → PostgreSQL migration](https://community.n8n.io/t/how-to-migrate-from-sqlite-to-postgresql/2170) — 15.4k views
- [Credentials lost after Docker restart](https://community.n8n.io/t/credentials-lost-after-docker-restart/226303)
- [File "x" is not writable](https://community.n8n.io/t/read-write-files-from-disk-the-file-x-is-not-writable/267765) — 1.2k views
- [SQLite crash — paid recovery](https://community.n8n.io/t/urgent-docker-hosted-n8n-on-sqlite-crashed-possible-db-corruption-looking-for-paid-expert-to-recover-data-and-migrate-to-postgres/257163)
- [Docker healthcheck request](https://community.n8n.io/t/docker-image-suggestion-add-healthcheck/31837) — 3.1k views
