# My webhook URL says `localhost:5678` and "Connection lost" keeps appearing

If any of these sound familiar:

- The webhook URL n8n shows ends in `localhost:5678` even though you reach n8n at a real domain
- A "Connection lost" banner appears every few seconds in the editor
- OAuth callback URLs point to `localhost:5678` and providers reject them
- The Test URL works but the Production URL never fires
- Webhooks worked locally but break behind nginx, Caddy, Traefik, or Cloudflare

…you're hitting the most common n8n self-hosting issue. This is fixable in about 5 minutes.

## Confirm it's this issue

In the n8n editor:

1. Open any workflow that has a **Webhook trigger** node
2. Look at the *Production URL* it displays
3. If that URL contains `localhost`, `127.0.0.1`, or a port that's different from how *you* reach n8n in your browser → it's this issue

OR — if you just see a recurring "Connection lost. You have a connection issue or the server is down." banner that never goes away → it's this issue (specifically the WebSocket part).

## Why it happens

n8n needs to know two things to display correct URLs:

1. **What's its public-facing URL?** — set via `N8N_HOST`, `N8N_PROTOCOL`, `N8N_PORT`, and `WEBHOOK_URL` env vars. If you don't set them, n8n introspects its own bind address (`0.0.0.0:5678` inside the container) and guesses `http://localhost:5678/`. That guess is almost always wrong in production.
2. **What's the original request the user sent?** — extracted from `X-Forwarded-*` headers your reverse proxy must forward. If those headers are missing or n8n doesn't trust them, the editor's WebSocket push channel never connects → "Connection lost" banner.

## Fix

### 1. Set these env vars on n8n

```yaml
environment:
  - N8N_HOST=workflows.example.com
  - N8N_PROTOCOL=https
  - N8N_PORT=443
  - WEBHOOK_URL=https://workflows.example.com/
  - N8N_EDITOR_BASE_URL=https://workflows.example.com/
  - N8N_PROXY_HOPS=1     # how many proxies are in front of n8n
```

Replace `workflows.example.com` with your real public hostname. Note the trailing slash on `WEBHOOK_URL`.

### 2. If you use nginx, add the WebSocket lines to your config

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl http2;
    server_name workflows.example.com;

    proxy_read_timeout 300s;
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
    }
}
```

The two lines that fix "Connection lost" specifically:

```nginx
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

If you use **Caddy** or **Traefik**, WebSocket upgrades work automatically and you don't need the lines above — just keep the env vars from step 1.

If you use **Cloudflare**, WebSockets are on by default but the free plan has a 100-second idle timeout. Long-running executions can be cut off. Either upgrade or proxy long-running calls through normal HTTP.

### 3. Restart n8n

```bash
docker compose down
docker compose up -d
```

Reload the editor and verify:

- No "Connection lost" banner for at least a minute
- Webhook trigger Production URL now shows your real domain
- OAuth credential redirect URL now shows your real domain

## Prevent it next time

- Always set `WEBHOOK_URL` and `N8N_HOST` before pointing customers at any URL n8n displays
- When putting n8n behind a new reverse proxy, test the editor for the "Connection lost" banner before going live
- Set `N8N_PROXY_HOPS` to match how many proxies sit in front of n8n (Cloudflare in front of nginx = 2)
- When debugging, run `docker logs n8n | head -50` after startup — n8n logs the public URL it thinks it has, so you can verify it matches what you expect

## Related forum threads

- [How to change localhost:5678 from webhook url?](https://community.n8n.io/t/how-to-change-localhost-5678-from-webhook-url/27033) — 39.2k views
- [Connection lost / server down banner](https://community.n8n.io/t/connection-lost-you-have-a-connection-issue-or-the-server-is-down-n8n-should-reconnect-automatically-once-the-issue-is-resolved/80999) — 40.8k views
- [OAuth callback URL is localhost:5678](https://community.n8n.io/t/oauth-callback-url-is-localhost-5678/2867) — 34.6k views
- [Nginx configuration](https://community.n8n.io/t/nginx-configuration/111) — 28.9k views

## Reproduction

Want to see the bug and the fix side-by-side? The full reproduction is at [`repros/01-webhook-url/`](../repros/01-webhook-url/) — clone, `docker compose up`, see it break, switch to the fixed compose file, see it work.
