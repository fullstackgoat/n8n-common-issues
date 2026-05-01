# REPRO — Repro 01

## Setup

```bash
cd repros/01-webhook-url
cp .env.example .env
docker compose up -d
```

Wait ~10 seconds for n8n to finish booting:

```bash
docker compose logs -f n8n | grep "Editor is now accessible"
# Press Ctrl-C once you see it
```

Open <http://localhost:8080>.

## Symptom 1 — "Connection lost" banner

**Observed:** within ~5 seconds of loading the editor, a banner appears at the top reading *"Connection lost. You have a connection issue or the server is down."* It cycles between connecting and disconnecting forever.

**Expected:** the editor should stay connected silently.

**Why we see it:** the broken `nginx.conf` proxies HTTP fine but doesn't forward the WebSocket `Upgrade` and `Connection` headers, so the editor's push-channel handshake fails. n8n falls back to polling and shows the banner.

## Symptom 2 — Webhook URL is wrong

1. In the editor, create a new workflow
2. Add a **Webhook** trigger node
3. Look at the displayed *Production URL*

**Observed:**

```
http://localhost:5678/webhook/<uuid>
```

**Expected:**

```
http://localhost:8080/webhook/<uuid>
```

**Why we see it:** with no `WEBHOOK_URL` / `N8N_HOST` / `N8N_PROTOCOL` / `N8N_PORT` environment variables set, n8n introspects its own bind address (`0.0.0.0:5678` inside the container) and emits that as the public URL. Customers then paste this URL into Stripe / GitHub / Calendly / etc. and the webhooks never arrive.

## Symptom 3 — OAuth callback URL is wrong

1. Settings → *Credentials* → *Create Credential* → *Google OAuth2 API*
2. Look at the *OAuth Redirect URL* shown

**Observed:** `http://localhost:5678/rest/oauth2-credential/callback`

**Expected:** `http://localhost:8080/rest/oauth2-credential/callback`

**Why we see it:** same root cause as Symptom 2.

## Symptom 4 — Production webhook fires from wrong URL

1. Activate a workflow with a Webhook trigger
2. Try to call the public URL displayed in the trigger
3. From outside the container, the call to `http://localhost:5678/webhook/...` will fail (port 5678 isn't published by `docker-compose.yml`, only port 8080 is)

The customer thinks the webhook node is broken. The node is fine — the URL it advertised is just wrong.

## Apply the fix

```bash
docker compose down
docker compose -f docker-compose.fixed.yml up -d
```

Reload <http://localhost:8080> and re-check Symptoms 1–4.

**Expected after fix:**

- No "Connection lost" banner
- Webhook URL: `http://localhost:8080/webhook/<uuid>`
- OAuth callback: `http://localhost:8080/rest/oauth2-credential/callback`
- Webhook fires successfully from the public URL

## Teardown

```bash
docker compose -f docker-compose.fixed.yml down -v
```
