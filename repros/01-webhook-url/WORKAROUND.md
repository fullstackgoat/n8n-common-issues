# WORKAROUND — Repro 01

A copy-pasteable fix you can apply to a customer's instance today.

## Step 1 — Set the four env vars

In your `docker-compose.yml` (or wherever you set env vars):

```yaml
environment:
  - N8N_HOST=workflows.example.com          # your public hostname, no protocol
  - N8N_PROTOCOL=https                      # http or https — what the user types
  - N8N_PORT=443                            # the public-facing port (443 for https, 80 for http)
  - WEBHOOK_URL=https://workflows.example.com/   # the public URL with trailing slash
  - N8N_EDITOR_BASE_URL=https://workflows.example.com/
```

> **Why all four?** `WEBHOOK_URL` covers webhooks specifically. `N8N_HOST` / `N8N_PROTOCOL` / `N8N_PORT` cover OAuth callbacks, form triggers, and other public URLs. Setting only one usually leaves a corner of the UI broken.

## Step 2 — If you're behind a reverse proxy, add this to your nginx config

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl http2;
    server_name workflows.example.com;

    # ... your TLS config ...

    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
    proxy_buffering off;
    client_max_body_size 32m;

    location / {
        proxy_pass http://n8n:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        $connection_upgrade;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;
    }
}
```

The two lines that fix "Connection lost":

```nginx
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

## Step 3 — Tell n8n to trust the proxy headers

If you have one proxy in front (just nginx):

```yaml
- N8N_PROXY_HOPS=1
```

If you have Cloudflare in front of nginx:

```yaml
- N8N_PROXY_HOPS=2
```

Without this, n8n won't trust `X-Forwarded-Proto` and will still emit `http://` URLs even though customers reach you via `https://`.

## Step 4 — Restart and verify

```bash
docker compose down
docker compose up -d
```

In the editor:

1. **No "Connection lost" banner** for at least 60 seconds → ✓ WebSocket fix works
2. Add a Webhook trigger → **Production URL shows your real domain** → ✓ env vars work
3. Open any OAuth credential → **Redirect URL shows your real domain** → ✓ same

## Per-platform notes

### Caddy
Caddy auto-handles WebSocket upgrades. Just `reverse_proxy n8n:5678` is enough. Still set the four env vars on n8n.

### Traefik
Auto-handles WebSocket upgrades. Still set the four env vars.

### AWS ALB
Enable "Web Sockets" on the target group. Set the listener idle timeout to ≥ 300s.

### Cloudflare
WebSockets are enabled by default. The catch: Cloudflare's free plan has a 100s idle timeout, so very long executions can get cut. If that bites you, either upgrade or proxy long-running calls through HTTP polling.

### ngrok (local dev)
```bash
ngrok http 5678
# then set:
WEBHOOK_URL=https://<your-tunnel>.ngrok-free.app/
N8N_HOST=<your-tunnel>.ngrok-free.app
N8N_PROTOCOL=https
```
The tunnel hostname changes every restart on the free plan. Add it to OAuth client allow-lists each time, or upgrade for a stable subdomain.

## When the customer's webhook still doesn't fire

After fixing this, if a webhook still doesn't fire, it's almost certainly one of:

- The provider hit *Test* URL instead of *Production* URL — see [Repro 05](../05-webhook-double-trigger/)
- The workflow isn't activated (toggle in upper-right is off)
- The provider expects a sync response and n8n is configured for async, or vice versa

Walk the customer through those in that order.
