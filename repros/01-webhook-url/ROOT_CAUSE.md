# ROOT CAUSE — Repro 01

## TL;DR

n8n self-introspects its bind address and emits it as the public-facing URL whenever you don't tell it otherwise. Reverse proxies further confuse the picture by stripping headers n8n needs to know "what was the original URL the user requested?". The fix is two parts: env vars (so n8n knows its public URL) **and** proxy headers (so n8n trusts what it sees).

## Layer 1 — n8n's URL-resolution logic

When n8n boots, it determines four things in this order:

1. **`N8N_HOST`** → the public hostname customers use to reach n8n. Default: `localhost`.
2. **`N8N_PROTOCOL`** → `http` or `https`. Default: `http`.
3. **`N8N_PORT`** → the public-facing port. Default: `5678`.
4. **`WEBHOOK_URL`** → an explicit override that wins over (1)+(2)+(3) for webhook URLs.

Anywhere the editor or REST API needs to print a URL the user will see — webhook trigger Production URL, OAuth callback URL, form trigger URL, public template share URL — those four values are concatenated.

If you don't set them, you get `http://localhost:5678/...` printed to a customer who's accessing n8n at `https://workflows.example.com/...`. The webhook node isn't broken; n8n simply doesn't know its own public URL.

**Reference:** see the [Server setup environment variables](https://docs.n8n.io/hosting/configuration/environment-variables/deployment/) page in n8n docs.

## Layer 2 — The reverse-proxy WebSocket handshake

n8n's editor uses a WebSocket-style push channel (originally socket.io, now polling-with-upgrade) for live execution updates and the multi-user editing indicator. The browser sends:

```
GET /rest/push HTTP/1.1
Host: workflows.example.com
Upgrade: websocket
Connection: Upgrade
```

A correctly configured nginx forwards `Upgrade` and `Connection` to the upstream. A *misconfigured* nginx (the default `proxy_pass` config in most online tutorials) silently drops them, downgrading the request to plain HTTP. The browser sees a non-101 response and reports the channel as disconnected. The editor displays *"Connection lost"* and retries indefinitely.

This is what the broken [`nginx.conf`](nginx.conf) reproduces, and what the [`nginx.fixed.conf`](nginx.fixed.conf) fixes.

The same problem appears in:

- **Caddy:** works out-of-the-box (Caddy auto-handles WebSocket upgrades). No fix required.
- **Traefik:** auto-handles. Fine by default.
- **AWS ALB / GCP LB / Azure App Gateway:** WebSocket support must be explicitly enabled on the listener/target group.
- **Cloudflare:** WebSockets are on by default for Free plan, but timeouts default to 100s — long-running executions get cut off. Set a longer keepalive.

## Layer 3 — `X-Forwarded-*` headers

When n8n is behind a proxy that terminates TLS, `req.protocol` inside Node.js sees `http` even though the user typed `https://`. Without the `X-Forwarded-Proto` header forwarded, n8n generates `http://` URLs even though it should be generating `https://`.

The fixed nginx config sets:

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
```

n8n trusts these headers as long as `N8N_PROXY_HOPS` matches the number of proxies in front of it (default 0 = trust no proxy). For a single nginx in front: `N8N_PROXY_HOPS=1`. For Cloudflare → nginx → n8n: `N8N_PROXY_HOPS=2`.

## Why this is the #1 self-hosting failure

Three reasons, all structural:

1. **The defaults are friendly to local development and hostile to production.** `localhost:5678` is the right answer for `docker run` on your laptop and the wrong answer for everyone else.
2. **The error mode is silent.** The webhook node renders a perfectly valid-looking URL. There's no warning that says *"this URL points to localhost; that's probably wrong."* The customer only finds out when downstream provider webhooks fail to arrive.
3. **Reverse-proxy tutorials online are usually incomplete.** The minimal `proxy_pass http://upstream` config that works for any normal Node.js app is *not* enough for n8n because of the WebSocket channel.

## Forum citations

- [How to change localhost:5678 from webhook url?](https://community.n8n.io/t/how-to-change-localhost-5678-from-webhook-url/27033) — 39.2k views — symptom 2
- [Connection lost / server down banner](https://community.n8n.io/t/connection-lost-you-have-a-connection-issue-or-the-server-is-down-n8n-should-reconnect-automatically-once-the-issue-is-resolved/80999) — 40.8k views — symptom 1
- [OAuth callback URL is localhost:5678](https://community.n8n.io/t/oauth-callback-url-is-localhost-5678/2867) — 34.6k views — symptom 3
- [Nginx configuration](https://community.n8n.io/t/nginx-configuration/111) — 28.9k views — proxy guidance from the community

## Upstream issue

Symptoms persist across multiple major versions because the env vars are documented but not enforced. A reasonable product-side improvement would be: *"n8n detected that `WEBHOOK_URL` is unset and your bind address resolves to localhost. Public URLs may be wrong. Set `WEBHOOK_URL` to suppress this warning."* — printed at startup and surfaced in the editor banner.
