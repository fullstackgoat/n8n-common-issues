# REPRO — Repro 03

Three deterministic symptoms, all reproducible 100% of the time on a clean Docker host. Each symptom is a routine action that customers perform daily — until the first time it eats their data.

## Setup

```bash
cd repros/03-docker-persistence
cp .env.example .env

# Generate and pin a real encryption key (only used by the fixed stack;
# the broken stack ignores it on purpose).
KEY=$(openssl rand -hex 32) && sed -i.bak "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=$KEY|" .env && rm .env.bak
```

## Symptom 1 — Credentials gone after `docker compose down`

The single most common forum thread in this cluster. User runs `docker compose down`, runs `up` again to apply a config change, all credentials are gone.

### Reproduce

```bash
# 1. Boot the broken stack
docker compose up -d
docker compose logs -f n8n | grep "Editor is now accessible"
# Press Ctrl-C once you see it
```

Open <http://localhost:5678>, create an owner account, then:

1. **Settings** → **Credentials** → **Create Credential**
2. Pick any type (e.g. *Header Auth*) and save it with a memorable name like `repro-credential`

```bash
# 2. Routine restart, the way every tutorial recommends
docker compose down
docker compose up -d
```

Reload <http://localhost:5678>. You'll be prompted to **create a new owner account**. The credential is gone. Every workflow you'd built is gone. The entire instance has been wiped.

**Observed:** New-owner setup screen. Database empty.

**Expected:** Editor reopens to your dashboard. Credential `repro-credential` is still listed.

**Why we see it:** The broken stack has no `volumes:` block. n8n writes everything to `/home/node/.n8n` *inside the container's writable layer*. `docker compose down` removes the container, and the writable layer goes with it. There's no warning — the second container looks identical to the first because the only difference is what's no longer there.

## Symptom 2 — Credentials unreadable after encryption-key drift

Even if you have a volume, the credentials are encrypted. If the key changes between runs, the data is still on disk but unreadable.

### Reproduce

```bash
# 3. Switch to the fixed stack, with a pinned encryption key.
docker compose down
docker compose -f docker-compose.fixed.yml up -d
```

Wait for `Editor is now accessible`, open <http://localhost:5678>, create an owner account, create a credential.

Now simulate the most common cause of key drift — a developer setting up a second environment without copying the key:

```bash
# Save the old key, then rotate to a new one (this is the bug)
OLD_KEY=$(grep '^N8N_ENCRYPTION_KEY=' .env | cut -d= -f2)
NEW_KEY=$(openssl rand -hex 32)
sed -i.bak "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=$NEW_KEY|" .env && rm .env.bak

docker compose -f docker-compose.fixed.yml down
docker compose -f docker-compose.fixed.yml up -d
```

Reload the editor and try to **edit** the credential you just created.

**Observed:** Either the credential fields appear empty / scrambled, or the workflow that uses it fails at runtime with an error like *"Could not decrypt credentials"*.

**Expected:** Credential opens normally; workflow runs.

**Why we see it:** n8n stores credentials AES-encrypted with `N8N_ENCRYPTION_KEY` as the key. The volume still has the encrypted blob, but the key needed to decrypt it is no longer the one n8n is booting with.

### Restore

```bash
sed -i.bak "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=$OLD_KEY|" .env && rm .env.bak
docker compose -f docker-compose.fixed.yml down
docker compose -f docker-compose.fixed.yml up -d
```

Credentials decrypt again. **Lesson: back up the key alongside the volume. They're a pair.**

## Symptom 3 — Healthcheck lies

Most tutorials skip the healthcheck. `docker compose ps` shows the service as healthy even when n8n is on fire.

### Reproduce — broken stack lies

```bash
docker compose -f docker-compose.fixed.yml down
docker compose up -d
sleep 2
docker compose ps
```

**Observed:**

```
NAME                                       STATUS
n8n-repro-03-docker-persistence-broken-n8n-1   Up 2 seconds
```

`Up 2 seconds`. No health information. The container is running, but n8n's editor takes ~10–20 seconds to actually be ready. During that window any reverse proxy / orchestration system hitting `/healthz` will get a connection refused, but Compose itself reports nothing wrong.

### The fixed stack tells the truth

```bash
docker compose down
docker compose -f docker-compose.fixed.yml up -d
sleep 5
docker compose -f docker-compose.fixed.yml ps
# … wait 30s …
docker compose -f docker-compose.fixed.yml ps
```

**Observed at t=5s:**

```
STATUS
Up 5 seconds (health: starting)
```

**Observed at t=35s:**

```
STATUS
Up 35 seconds (healthy)
```

Now `docker compose ps` matches reality. Watchdogs, load balancers, and CI scripts can trust it.

**Why we see it:** A container without a `healthcheck:` block defaults to *no* health information — which Docker reports as "absent" but most tools (and most humans) read as "healthy." Adding `healthcheck:` against `/healthz` makes Docker actively probe the editor and degrade the status when it's not responding.

## Apply the fix end-to-end

```bash
docker compose down -v                                       # nuke broken
docker compose -f docker-compose.fixed.yml up -d             # boot fixed
docker compose -f docker-compose.fixed.yml logs -f n8n | grep "Editor is now accessible"
```

Now repeat Symptom 1's steps with the fixed stack:

1. Create an owner account, create a credential
2. `docker compose -f docker-compose.fixed.yml down`
3. `docker compose -f docker-compose.fixed.yml up -d`
4. Reload — **owner account is still there, credential is still there.**

The volume `n8n_data` survives `down`; the encryption key in `.env` survives the container; the healthcheck reports the truth.

## Teardown

```bash
docker compose -f docker-compose.fixed.yml down -v
```

`-v` is required to remove the named volume. Without it the volume persists across this whole repro and the next.
