# Repro 01 — Webhook URL & "Connection lost"

The single most common self-hosting onboarding failure on `community.n8n.io`.

## Symptoms (any combination)

- The "Test URL" works but the "Production URL" doesn't fire
- Webhook URLs in the UI show `http://localhost:5678/webhook/...` even though n8n is on a public domain
- "Connection lost" banner appears every few seconds and never recovers
- OAuth callbacks redirect to `localhost:5678` instead of your real hostname
- Webhooks work locally but fail behind nginx, Caddy, Traefik, or Cloudflare

## What this repro proves

That all of the symptoms above stem from a small set of misconfigurations around `WEBHOOK_URL`, `N8N_HOST`, `N8N_PROTOCOL`, and your reverse proxy's WebSocket headers — and that fixing them in the right order makes every symptom disappear.

## Files

| File | Purpose |
|---|---|
| [`docker-compose.yml`](docker-compose.yml) | Boots n8n + nginx in a configuration that reproduces the bug (intentional misconfig) |
| [`docker-compose.fixed.yml`](docker-compose.fixed.yml) | The same stack with the fix applied — boot this to verify |
| [`.env.example`](.env.example) | All required env vars, commented |
| [`nginx.conf`](nginx.conf) | The reverse proxy config; comments mark the WebSocket-handling lines |
| [`REPRO.md`](REPRO.md) | Step-by-step reproduction with observed vs expected output |
| [`ROOT_CAUSE.md`](ROOT_CAUSE.md) | Why each symptom happens, with citations |
| [`WORKAROUND.md`](WORKAROUND.md) | The exact env-var/proxy changes a customer can apply right now |

## Quick start

```bash
cd repros/01-webhook-url
cp .env.example .env

# 1) Boot the broken stack
docker compose up -d
# Visit http://localhost:8080 — note the symptoms in REPRO.md

# 2) Boot the fixed stack
docker compose down
docker compose -f docker-compose.fixed.yml up -d
# Visit http://localhost:8080 — symptoms gone
```

## Linked KB article

[`kb/01-webhook-url.md`](../../kb/01-webhook-url.md) — the customer-facing fix you can paste into a ticket reply.

## Source threads

- [How to change localhost:5678 from webhook url?](https://community.n8n.io/t/how-to-change-localhost-5678-from-webhook-url/27033) — 39.2k views
- [Connection lost / server down banner](https://community.n8n.io/t/connection-lost-you-have-a-connection-issue-or-the-server-is-down-n8n-should-reconnect-automatically-once-the-issue-is-resolved/80999) — 40.8k views
- [Connection lost (legacy)](https://community.n8n.io/t/connection-lost/1471) — 21.8k views
- [Nginx configuration](https://community.n8n.io/t/nginx-configuration/111) — 28.9k views
- [OAuth callback URL is localhost:5678](https://community.n8n.io/t/oauth-callback-url-is-localhost-5678/2867) — 34.6k views
