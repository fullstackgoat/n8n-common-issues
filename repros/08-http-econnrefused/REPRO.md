# REPRO — Repro 08

Three scenarios, two stacks each. Each scenario reproduces 100% of the time on a clean Docker host. The same workflow JSONs are used on both broken and fixed stacks — only the n8n container's environment differs between them.

## Setup

```bash
cd repros/08-http-econnrefused
cp .env.example .env
```

> If you already have a real Ollama running on this machine, port 11434 will collide with the `fake-ollama` container. Either stop your local Ollama for the duration of the repro, or change `11434:80` to a free port in both compose files (and the URLs in `workflows/1-ollama-localhost.json`).

```bash
# Boot the broken stack
docker compose up -d
docker compose logs -f n8n | grep "Editor is now accessible"
# Press Ctrl-C once you see it
```

Open <http://localhost:5678>, create an owner account, then for each scenario below: **import the named workflow JSON**, click **Test workflow** at the bottom of the canvas, and inspect each HTTP node's output panel.

## Scenario 1 — `localhost` from inside the container

Import [`workflows/1-ollama-localhost.json`](workflows/1-ollama-localhost.json) and click **Test workflow**. The workflow has two HTTP Request nodes hitting the same target via two different URLs.

### On the broken stack

| Node | URL | Result |
|---|---|---|
| `A — http://localhost:11434/` | n8n's own localhost (the n8n container itself) | ✗ `Error: connect ECONNREFUSED ::1:11434` *(or `127.0.0.1:11434` on older Node — same thing, just IPv6 vs IPv4 loopback)* |
| `B — http://host.docker.internal:11434/` | the Docker host's network | ✓ on **Docker Desktop (macOS / Windows)** — `host.docker.internal` is mapped automatically; ✗ on **Linux** — `Error: getaddrinfo ENOTFOUND host.docker.internal` |

**Why we see it:** Inside the n8n container, `localhost` resolves to the container's own loopback (Node 22+ prefers `::1`, the IPv6 loopback, when both families are available). Nothing is listening on port 11434 there. The `fake-ollama` container is only reachable through Docker DNS as `fake-ollama` or — via the published `11434:80` mapping — through the host's network (`host.docker.internal`).

**The "works on my machine" trap:** Docker Desktop on macOS / Windows pre-populates `host.docker.internal` automatically, so node B succeeds there even on the broken stack. On bare Linux Docker, it fails — because `extra_hosts` isn't set in `docker-compose.yml`. This is the most common shape of *"it worked when the developer tested locally and broke when we deployed to the Linux production host."*

To force the Linux failure on Docker Desktop for testing, you can add `network_mode: bridge` and `dns: ["127.0.0.11"]` constraints — but the simpler honest test is *"deploy to a Linux host and watch node B start failing"*. The fix on the fixed stack (`extra_hosts: ["host.docker.internal:host-gateway"]`) is harmless on Docker Desktop and required on Linux, so set it everywhere and forget about it.

### After applying the fix

```bash
docker compose down
docker compose -f docker-compose.fixed.yml up -d
docker compose -f docker-compose.fixed.yml logs -f n8n | grep "Editor is now accessible"
# Press Ctrl-C once you see it
```

Re-import the same workflow (sign in to the new owner account first) and click **Test workflow** again.

| Node | URL | Result |
|---|---|---|
| `A — http://localhost:11434/` | still wrong | ✗ `connect ECONNREFUSED 127.0.0.1:11434` |
| `B — http://host.docker.internal:11434/` | resolves via `extra_hosts` | ✓ 200 OK; body shows `Hostname: <fake-ollama-container-id>`, `Name: fake-ollama` |

**Why node A still fails:** `localhost` inside the container is *always* the container itself, regardless of `extra_hosts`. The fix is two-part: env config (`extra_hosts: ["host.docker.internal:host-gateway"]` on n8n) **and** workflow URL change (`localhost` → `host.docker.internal`). Node A proves this — the URL itself is the bug.

## Scenario 2 — Internal API behind a forward proxy

Import [`workflows/2-corp-proxy.json`](workflows/2-corp-proxy.json) and click **Test workflow**.

### On the broken stack

| Node | URL | Result |
|---|---|---|
| `GET http://internal-api/` | a service on the `private` network only | ✗ `Error: getaddrinfo ENOTFOUND internal-api` |

**Why we see it:** The `internal-api` container is on the `private` Docker network only. The `n8n` container is on `public` only. Docker's embedded DNS does not expose container names across separate networks, so the lookup fails. There is no route from `n8n` to `internal-api` directly.

### After applying the fix

(If you've already switched to the fixed stack from Scenario 1, just re-import this workflow. If you haven't, switch now.)

```bash
docker compose down
docker compose -f docker-compose.fixed.yml up -d
```

Re-import and run.

| Node | URL | Result |
|---|---|---|
| `GET http://internal-api/` | proxied via squid | ✓ 200 OK; body shows the request was forwarded |

**Why we see it:** The fixed stack sets `HTTP_PROXY=http://squid:3128` (and `HTTPS_PROXY=...`, `NO_PROXY=...`) on the n8n container. Node.js' standard HTTP libraries respect those env vars (via `proxy-from-env` semantics), so n8n's outbound request to `internal-api` is forwarded to Squid. Squid sits on **both** networks and can resolve and reach `internal-api`. The response is relayed back to n8n.

To prove the request really went through Squid, look in the response body for:

```
RemoteAddr: 172.x.x.x      ← squid's IP, not n8n's
```

That confirms `internal-api` saw Squid as the client, not n8n.

## Scenario 3 — HTTPS endpoint with a self-signed certificate

Import [`workflows/3-self-signed.json`](workflows/3-self-signed.json) and click **Test workflow**.

### On the broken stack

| Node | URL | Result |
|---|---|---|
| `GET https://self-signed/` | nginx + cert generated by `cert-init` | ✗ `NodeSslError: SSL Issue: consider using the 'Ignore SSL issues' option` |

**Why we see it:** Node.js verifies TLS certificates against the system CA bundle. The cert that `cert-init` generated and `self-signed-nginx` is serving was signed by itself — there's no chain of trust to a CA in the bundle. The handshake aborts before any HTTP data is sent.

n8n's HTTP Request node intercepts TLS errors and surfaces them with the *"consider using the 'Ignore SSL issues' option"* hint instead of the raw Node.js error. The underlying error is still one of `self signed certificate`, `unable to verify the first certificate`, `SELF_SIGNED_CERT_IN_CHAIN`, or `DEPTH_ZERO_SELF_SIGNED_CERT` (depends on Node version and cert chain depth). The "Ignore SSL issues" toggle the hint references is on the HTTP node's own *Options → Ignore SSL Issues*, which is the per-node version of `NODE_TLS_REJECT_UNAUTHORIZED=0` — convenient but with the same tradeoffs (see [`WORKAROUND.md`](WORKAROUND.md)).

### After applying the fix

(Same as Scenario 2 — switch to the fixed stack if you haven't already.)

```bash
docker compose down
docker compose -f docker-compose.fixed.yml up -d
```

Re-import and run.

| Node | URL | Result |
|---|---|---|
| `GET https://self-signed/` | trusted via `NODE_EXTRA_CA_CERTS` | ✓ 200 OK; body is `{"server":"self-signed-nginx","tls":"self-signed","status":"ok"}` |

**Why we see it:** The fixed stack mounts the same `certs` named volume that `cert-init` wrote to (read-only) and sets `NODE_EXTRA_CA_CERTS=/certs/server.crt`. Node.js reads that env var at startup and adds the listed cert to its in-process trust store. Verification of any cert signed by it now succeeds — without disabling TLS verification globally.

## Apply the fix end-to-end (all three scenarios in one cycle)

```bash
docker compose down -v                                       # nuke broken
docker compose -f docker-compose.fixed.yml up -d             # boot fixed
docker compose -f docker-compose.fixed.yml logs -f n8n | grep "Editor is now accessible"
```

Sign in (or create a fresh owner account on the new instance), re-import all three workflows, click **Test workflow** on each. Expected after fix:

| Workflow | Result |
|---|---|
| `1-ollama-localhost.json` | Node A ✗ `ECONNREFUSED ::1:11434` (URL still wrong, by design); node B ✓ 200 OK with `Name: fake-ollama` |
| `2-corp-proxy.json` | ✓ 200 OK; response body shows `RemoteAddr` as the squid container's IP, not n8n's (proves the request was proxied) |
| `3-self-signed.json` | ✓ 200 OK; body is `{"server":"self-signed-nginx","tls":"self-signed","status":"ok"}`. TLS verification stays on for everything else (compare with `Ignore SSL Issues`, which would have disabled it globally) |

## Teardown

```bash
docker compose -f docker-compose.fixed.yml down -v
```

`-v` removes the `n8n_data` and `certs` named volumes. Without it, the cert generated by `cert-init` persists across full repro cycles (which is fine — `cert-init` skips regeneration on subsequent boots).
