# WORKAROUND — Repro 03

A copy-pasteable persistence stack you can hand to a customer today, plus the SQLite → Postgres migration most production deployments outgrow into.

## Step 1 — Generate and pin an encryption key

```bash
openssl rand -hex 32
# 8f3c1a...64-hex-chars
```

Store the output somewhere safe (your secrets manager, a sealed envelope, your password manager). Treat it like a database backup — losing it means losing every credential.

In your `.env`:

```bash
N8N_ENCRYPTION_KEY=8f3c1a...64-hex-chars
```

In your compose file:

```yaml
environment:
  - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
```

## Step 2 — Use a named volume, never a writable-layer write

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

`docker compose down` is now safe. Only `docker compose down -v` removes the volume.

If you specifically need files visible on the host (for a backup script that reads them directly), use a bind-mount **and** match UID:

```yaml
services:
  n8n:
    user: "1000:1000"
    volumes:
      - ./n8n-data:/home/node/.n8n
```

```bash
mkdir -p ./n8n-data && sudo chown -R 1000:1000 ./n8n-data
```

## Step 3 — Add a real healthcheck

```yaml
healthcheck:
  test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:5678/healthz || exit 1"]
  interval: 30s
  timeout: 5s
  retries: 5
  start_period: 30s
```

`docker compose ps` now reports `(healthy)` / `(unhealthy)` instead of just `Up Xs`. Wire your watchdog (Uptime Kuma, healthchecks.io, the `restart: on-failure` policy in another stack) against the real signal.

## Step 4 — Back up the right things, on a schedule

The complete backup set for a SQLite-backed n8n is **three things**:

1. The `n8n_data` volume (or its bind-mount equivalent)
2. The `N8N_ENCRYPTION_KEY` env var value
3. A copy of `docker-compose.yml` and `.env` (so you can rebuild the stack)

A nightly script (`crontab -e` on the host):

```bash
#!/usr/bin/env bash
set -euo pipefail

STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR=/srv/n8n-backups
mkdir -p "$BACKUP_DIR"

# Tar the named volume by reading it through a one-shot busybox container
docker run --rm \
  -v n8n_data:/data:ro \
  -v "$BACKUP_DIR":/backup \
  busybox tar czf "/backup/n8n-data-$STAMP.tar.gz" -C /data .

# Stash the encryption key alongside (it's already in .env on disk, but be explicit)
grep '^N8N_ENCRYPTION_KEY=' /opt/n8n/.env > "$BACKUP_DIR/n8n-key-$STAMP.env"

# Rotate: keep last 14 days
find "$BACKUP_DIR" -type f -mtime +14 -delete
```

To restore on a fresh host:

```bash
# 1. Lay down docker-compose.yml + .env (with the same N8N_ENCRYPTION_KEY)
# 2. Create the volume but don't boot n8n yet
docker volume create n8n_data
docker run --rm \
  -v n8n_data:/data \
  -v /path/to/backup:/backup:ro \
  busybox tar xzf /backup/n8n-data-YYYYMMDD-HHMMSS.tar.gz -C /data
# 3. docker compose up -d
```

## Step 5 — Migrate SQLite → Postgres (when you outgrow single-instance)

You've outgrown SQLite when **any** of these are true:

- You're running queue mode with one or more workers (SQLite can't be shared across processes safely)
- You see *"database is locked"* errors under concurrent load
- Your `database.sqlite` file is over ~1 GB
- You're moving to a multi-host or HA setup

n8n ships an official migration path. The full procedure:

### 5a. Add Postgres to your stack

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${DB_POSTGRESDB_USER}
      - POSTGRES_PASSWORD=${DB_POSTGRESDB_PASSWORD}
      - POSTGRES_DB=${DB_POSTGRESDB_DATABASE}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  n8n:
    # ... your existing n8n service ...
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      # Add these to whatever you already have:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${DB_POSTGRESDB_DATABASE}
      - DB_POSTGRESDB_USER=${DB_POSTGRESDB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}

volumes:
  n8n_data:
  postgres_data:
```

### 5b. Export from SQLite

With the **old** stack still configured for SQLite:

```bash
docker compose exec n8n n8n export:credentials --backup --output=/home/node/.n8n/backup/credentials/
docker compose exec n8n n8n export:workflow     --backup --output=/home/node/.n8n/backup/workflows/
```

The `--backup` flag exports each entity as its own JSON file under the output directory.

### 5c. Switch to Postgres and import

```bash
docker compose down                              # stop, but DON'T -v
# Edit .env: add DB_TYPE=postgresdb and the four DB_POSTGRESDB_* vars
docker compose up -d                             # boots n8n + postgres; n8n auto-creates its schema
docker compose logs -f n8n | grep "Editor is now accessible"
docker compose exec n8n n8n import:credentials --separate --input=/home/node/.n8n/backup/credentials/
docker compose exec n8n n8n import:workflow     --separate --input=/home/node/.n8n/backup/workflows/
```

Reload the editor. All workflows and credentials should be present, now backed by Postgres.

### 5d. Verify, then archive SQLite

```bash
# Sanity check from inside the container
docker compose exec postgres psql -U $DB_POSTGRESDB_USER -d $DB_POSTGRESDB_DATABASE -c \
  "SELECT count(*) FROM workflow_entity; SELECT count(*) FROM credentials_entity;"
```

Once the counts match what you had in SQLite, archive `/home/node/.n8n/database.sqlite` somewhere safe and leave it there. n8n will ignore it now that `DB_TYPE=postgresdb` is set, but keep it as a fall-back for ~30 days before deleting.

### Common migration gotchas

- **Encryption key.** `N8N_ENCRYPTION_KEY` must be the same value during export and import. Otherwise the imported credentials decrypt to garbage.
- **Execution history is not exported by default.** `n8n export:credentials` and `export:workflow` cover credentials and workflows. Execution data lives in a separate table and is normally not migrated. If you need it, dump the SQLite `execution_entity` table directly with `sqlite3 database.sqlite ".dump execution_entity"` and replay against Postgres — but expect schema drift between versions.
- **Tags and credential sharing.** Tags are exported. Per-user credential sharing (n8n Cloud / Enterprise) is not always preserved across migrations; re-share after import.
- **Run import on a quiet system.** Pause webhook providers and disable cron triggers during the cutover, or you'll get duplicate runs while the import is in flight.

## Per-platform notes

### Linux (Ubuntu / Debian / Fedora)

- Default user is usually UID `1000`. Named volumes Just Work.
- For bind-mounts, `sudo chown -R 1000:1000 ./n8n-data` once is enough.
- If you're on **rootless Docker**, the in-container `1000` maps to a high subuid (e.g. `100999`). Either match that or use a named volume (recommended).

### macOS / Windows (Docker Desktop)

- Named volumes are fine. Don't bind-mount unless you have a reason — the virtualized filesystem is slower and UID translation is opaque.
- Docker Desktop's "Resources → File Sharing" must include any path you do bind-mount.

### Synology / TrueNAS / NAS appliances

- The container manager often runs containers as a non-1000 user by default. Set `user:` explicitly.
- Don't bind-mount to a network share (NFS / SMB) for the SQLite DB — the locking semantics break it. Use a local volume and replicate at the storage layer if you need redundancy.

### Kubernetes

- Use a `PersistentVolumeClaim`, not an `emptyDir`. The `emptyDir` lifecycle matches the pod, not the deployment.
- `securityContext.runAsUser: 1000` on the pod spec is the equivalent of `user: "1000:1000"` in compose.
- For the encryption key, use a `Secret` mounted as an env var. Never bake it into the image.

## When the customer's data still seems gone

After applying the fix, if a customer says data is still missing:

1. **Did they run `docker compose down -v` (with `-v`)?** That removes the volume. Look in `docker volume ls` — if `n8n_data` isn't there, it's been wiped.
2. **Did they change `N8N_ENCRYPTION_KEY`?** Check git history of `.env`. If the key changed, credentials won't decrypt even though the workflows are still listed.
3. **Did they migrate to a new compose project name?** The `name:` directive at the top of compose files namespaces volumes. Switching project names creates a fresh `n8n_data` volume and leaves the old one stranded — visible in `docker volume ls` as `<oldname>_n8n_data`.
4. **Is it a UID/GID `EACCES`?** `docker compose logs n8n | grep -i "permission\|EACCES"`. Fix the bind-mount owner or switch to a named volume.

That sequence covers ~95% of "I lost my data" tickets in this cluster.
