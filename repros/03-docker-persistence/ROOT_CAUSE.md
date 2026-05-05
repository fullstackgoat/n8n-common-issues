# ROOT CAUSE — Repro 03

## TL;DR

n8n's data layer is one directory: `/home/node/.n8n` inside the container. Three things must be true for it to survive a routine restart: that directory must be on a **named volume** (not the container's writable layer), the **`N8N_ENCRYPTION_KEY`** that protects it must be pinned outside the volume, and the **UID/GID** the container writes as must match the file ownership on the volume. Miss any one and the next `docker compose down` is destructive — silently.

## Layer 1 — The container writable layer is ephemeral

Every Docker container has a thin writable layer on top of its read-only image layers. By default, n8n writes its SQLite database, `config` file (which contains the auto-generated encryption key), and binary-data attachments into `/home/node/.n8n` — and *if you don't mount a volume there*, that path lives in the writable layer.

`docker compose down` removes the container. The writable layer goes with it. There is no warning, no `--data-loss` flag, no confirmation prompt. The next `docker compose up` creates a fresh container from the same image, which now has an empty `/home/node/.n8n` because every file in there came from the previous container's writable layer.

This is the modal n8n forum complaint in this cluster — *["Credentials lost after Docker restart"](https://community.n8n.io/t/credentials-lost-after-docker-restart/226303)*. The user sees a "create owner account" screen and concludes that n8n broke. n8n didn't break; the data was never persisted in the first place.

The fix is one line:

```yaml
volumes:
  - n8n_data:/home/node/.n8n
```

Plus a top-level `volumes: { n8n_data: }`. Now `/home/node/.n8n` lives in a named volume that's independent of the container lifecycle. `docker compose down` leaves the volume intact; only `docker compose down -v` removes it.

## Layer 2 — `N8N_ENCRYPTION_KEY` is the single point of failure

n8n encrypts every credential with AES, using `N8N_ENCRYPTION_KEY` as the key. When that env var is unset on first boot, n8n generates a random key and stores it in `/home/node/.n8n/config`:

```json
{ "encryptionKey": "f8a3...64-hex-chars" }
```

This is convenient — the user doesn't have to set anything to get started. It's also a footgun:

- If you rebuild the volume (snapshot restore, migration to a new host, accidental `down -v`), the new `config` file has a new auto-generated key. Existing credential blobs on disk become unreadable.
- If you set `N8N_ENCRYPTION_KEY` *after* n8n has already auto-generated one, n8n trusts the env var and abandons the on-disk one. Existing credentials become unreadable.
- If the volume mounts cleanly but the env var is set to a different value than the on-disk one, n8n trusts the env var. Existing credentials become unreadable.

In every case, the *encrypted credential blobs are still on disk*. The user can see them in the SQLite `credentials_entity` table. They just can't be decrypted, because the AES key the encryption was performed with is gone.

The fix is also one line:

```yaml
environment:
  - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
```

…with a value generated once via `openssl rand -hex 32` and treated as a backup artifact in its own right. **The volume and the key must travel together.** A volume snapshot without the key it was encrypted under is useless.

## Layer 3 — UID/GID and the bind-mount EACCES trap

`docker.n8n.io/n8nio/n8n` runs as the `node` user (UID `1000`, GID `1000`) inside the container. When you mount a **named volume**, Docker creates the volume with whatever ownership the container's first write requires — `1000:1000` — and writes work fine.

When you mount a **host bind-mount** (e.g. `./n8n-data:/home/node/.n8n`), Docker does *not* magically remap UIDs:

- On **Linux**, the bind-mount inherits the host directory's owner. If you created `./n8n-data` as your user (typically also UID `1000` on Ubuntu / Debian / Fedora) you'll get away with it. If you created it as root via `sudo mkdir`, the container's `node` user (`1000`) can't write to it and every workflow that touches `/home/node/.n8n` (which is *every workflow*) fails with `EACCES: permission denied`.
- On **macOS / Windows Docker Desktop**, the host filesystem is virtualized. UID translation usually papers over the problem for the SQLite DB, but workflows that try to write binary data to a host path *outside* `/home/node/.n8n` (e.g. `/data/output.txt` mounted from the host) hit it.
- On **rootless Docker**, the user namespace remapping makes the in-container `1000` map to a high host UID like `100999`. Bind-mounts owned by your host user become unwritable.

The forum thread *["Read/Write Files from Disk: The file 'x' is not writable"](https://community.n8n.io/t/read-write-files-from-disk-the-file-x-is-not-writable/267765)* is the canonical version of this. Fix is two lines:

```yaml
user: "1000:1000"   # match the host owner of any bind-mount you use
```

…and on the host:

```bash
sudo chown -R 1000:1000 ./n8n-data
```

Named volumes sidestep the whole class of problem; bind-mounts are only worth the pain when you specifically need files visible on the host filesystem (e.g. backup scripts that read `/home/node/.n8n` directly).

## The healthcheck is not the bug, but it hides the bug

A container that boots cleanly, accepts the volume mount, has the right encryption key, and then crashes 30 seconds later is reported as *"Up 30 seconds"* — same as a container that's been serving traffic for 30 seconds. Without `healthcheck:`, your watchdog has no signal to act on.

n8n exposes `/healthz` on the editor port. Probing it every 30 seconds gives `docker compose ps` real status (`healthy` / `unhealthy` / `starting`) and lets `restart: unless-stopped` actually restart on failure instead of restarting on crash-loop only.

This isn't strictly part of the persistence story, but every "I lost my data" forum thread starts with a user who believed their stack was fine because Compose said so. Adding the healthcheck doesn't prevent data loss; it makes the wrongness visible at the right moment.

## Why this is the #1 self-hosting failure cluster

Three structural reasons:

1. **The Docker `run` command in the n8n quickstart docs uses an anonymous volume**, which is technically persistent but invisible (no name, no `docker volume ls` entry the user recognizes). Users who later switch to `docker-compose` often omit the `volumes:` block entirely because the quickstart didn't make it look load-bearing.
2. **The encryption key is silently auto-generated.** A configuration that "just works" on day one is the most dangerous kind, because the user doesn't know there's a key to back up. The first time they find out is during disaster recovery.
3. **Docker's `down` command doesn't distinguish "stop containers" from "destroy state."** Users coming from other process managers (systemd, pm2, supervisord) don't expect `down` to be destructive. Many tutorials say *"if it's broken, run `docker compose down && up`"* — which works for stateless services and is catastrophic here.

## Forum citations

- [Credentials lost after Docker restart](https://community.n8n.io/t/credentials-lost-after-docker-restart/226303) — symptom 1 — "I restarted and everything is gone"
- [How to upgrade n8n with docker-compose](https://community.n8n.io/t/how-to-upgrade-n8n-with-docker-compose-setup/660) — 25k views — upgrade procedure that destroys data when run against a missing volume
- [Read/Write Files from Disk: The file "x" is not writable](https://community.n8n.io/t/read-write-files-from-disk-the-file-x-is-not-writable/267765) — 1.2k views — symptom 3 (UID mismatch on bind-mounts)
- [SQLite crash — paid recovery](https://community.n8n.io/t/urgent-docker-hosted-n8n-on-sqlite-crashed-possible-db-corruption-looking-for-paid-expert-to-recover-data-and-migrate-to-postgres/257163) — what happens when SQLite is used past its concurrency ceiling and the persistence stack hasn't been audited
- [Docker image suggestion: add healthcheck](https://community.n8n.io/t/docker-image-suggestion-add-healthcheck/31837) — 3.1k views — community-requested fix for the healthcheck-lies problem; still not in the upstream image
- [How to migrate from SQLite to PostgreSQL](https://community.n8n.io/t/how-to-migrate-from-sqlite-to-postgresql/2170) — 15.4k views — the next step after this repro

## Upstream issue

The single highest-leverage product fix would be: **on first boot, if `/home/node/.n8n` is on the writable layer (i.e. no volume mounted), n8n logs a giant warning and refuses to start unless an explicit `N8N_ALLOW_EPHEMERAL_STORAGE=true` is set.** Today, the failure is silent and only manifests on the *second* boot — by which time the user has already trusted it with real data. Failing loudly on the first boot would deflect a measurable fraction of the data-loss tickets in this cluster, and the cost is one log line and one env var.

Same logic for `N8N_ENCRYPTION_KEY`: if it's unset, log a warning at boot that says *"Encryption key auto-generated and stored at /home/node/.n8n/config. Back up this file alongside your volume — losing it makes credentials unrecoverable."* Today, the warning doesn't exist; the user finds out from a forum reply.
