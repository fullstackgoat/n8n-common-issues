# WORKAROUND — Repro 08

A copy-pasteable fix you can apply to a customer's instance today, broken out per scenario. All three fixes can coexist on the same n8n container without conflict.

## Scenario 1 — `localhost` from inside a container

The customer's symptom is one of:

```
Error: connect ECONNREFUSED 127.0.0.1:11434
Error: connect ECONNREFUSED ::1:11434
Error: getaddrinfo ENOTFOUND host.docker.internal      ← Linux only
```

The fix is **two parts**: an env change on the n8n container, and a URL change in the workflow.

### Part 1 — Add the host-gateway mapping (Linux only — harmless on macOS / Windows)

```yaml
services:
  n8n:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

`host-gateway` is a magic string Docker resolves to the host's gateway IP. On Docker Desktop (macOS, Windows) this mapping already exists by default, so the line is a no-op there. Setting it everywhere is the right default.

### Part 2 — Change the workflow URL

```diff
- http://localhost:11434/api/generate
+ http://host.docker.internal:11434/api/generate
```

`localhost` from inside the container is *always* the container itself, regardless of `extra_hosts`. The URL has to change.

### Per-platform notes

| Platform | What works |
|---|---|
| Docker Desktop (macOS) | `host.docker.internal` works out of the box; `extra_hosts` is a harmless no-op |
| Docker Desktop (Windows) | Same as macOS |
| Linux Docker (rootful) | `extra_hosts: ["host.docker.internal:host-gateway"]` required |
| Linux Docker (rootless) | `extra_hosts` works, but the host-gateway IP is the rootless network's gateway, not the host's `127.0.0.1`. If your service binds to `127.0.0.1` only, it won't be reachable. Bind to `0.0.0.0` or use a real LAN IP. |
| Kubernetes | `host.docker.internal` is not available. Either deploy the service inside the cluster, or use `hostNetwork: true` on the pod (with all the security caveats), or expose the host service via `Service` of type `ExternalName`. |

### When `host.docker.internal` still doesn't work

- The host service might be bound to `127.0.0.1` only. Confirm with `ss -tlnp | grep 11434` on the host. If it shows `127.0.0.1:11434` and not `0.0.0.0:11434` or `*:11434`, the service refuses connections from the Docker bridge. For Ollama specifically, set `OLLAMA_HOST=0.0.0.0` and restart it.
- A host firewall might be blocking the Docker bridge. On Linux: `sudo ufw allow from 172.17.0.0/16` (or wherever Docker's bridge subnet is). On macOS: System Settings → Network → Firewall, add an allow rule.
- A corporate VPN can override DNS and route. Disconnect the VPN to test.

## Scenario 2 — HTTP proxy for outbound traffic

Symptom is one of:

```
Error: getaddrinfo ENOTFOUND <internal-hostname>
Error: connect ETIMEDOUT <ip>:<port>
Error: connect ECONNREFUSED 127.0.0.1:80    ← user pointed at a non-existent local proxy
```

### The fix — set the standard env vars on the n8n container

```yaml
services:
  n8n:
    environment:
      - HTTP_PROXY=http://proxy.example.com:3128
      - HTTPS_PROXY=http://proxy.example.com:3128
      - NO_PROXY=localhost,127.0.0.1,.svc.cluster.local,internal.example.com,10.0.0.0/8
```

Notes:

- **Both `HTTP_PROXY` and `HTTPS_PROXY` need to be set.** Most workflows use `https://` URLs; setting only `HTTP_PROXY` won't catch them.
- **`NO_PROXY` is mandatory in any non-trivial setup.** Without it, *every* outbound request — including DNS-resolvable container peers and `localhost` — round-trips through the proxy. That's slow and often broken (most corp proxies refuse to relay to internal hostnames).
- **The proxy URL must include the scheme** (`http://...`). Bare `proxy.example.com:3128` is not parsed correctly by all Node libraries.
- **If the proxy needs auth**: `HTTP_PROXY=http://user:pass@proxy.example.com:3128` works. URL-encode special characters in the password (`@` becomes `%40`, etc.). Better, use a credential injection mechanism your secrets manager supports.

### When n8n is itself the proxy ingress (Cloudflare, ngrok, etc.)

Don't set `HTTP_PROXY` on n8n in this case. The proxy is in front of n8n for *inbound* traffic, not in front of it for *outbound* traffic. Setting `HTTP_PROXY` here will route every outbound n8n request through the inbound proxy, which is wrong.

### Per-platform notes

| Platform | Quirk |
|---|---|
| docker-compose | Set in the `environment:` block — straightforward |
| Plain `docker run` | `-e HTTP_PROXY=... -e HTTPS_PROXY=... -e NO_PROXY=...` |
| Kubernetes | Set as env on the pod spec, OR as a `ConfigMap` mounted as env. **Don't** use the cluster-wide proxy via `containerd`/`dockerd` settings — n8n is a Node process and reads the env vars directly |
| systemd-managed n8n on bare metal | `Environment=HTTP_PROXY=...` in the unit file. After change: `systemctl daemon-reload && systemctl restart n8n` |

### Verifying the proxy is actually being used

After applying the fix, run a workflow that hits any external URL and look at the response headers / body. If the upstream sees a `Via:` header or a `RemoteAddr` matching your proxy's IP, the env vars are working. If the upstream sees n8n's container IP, n8n is bypassing the proxy.

## Scenario 3 — Self-signed or internal-CA cert

Symptom is one of:

```
Error: self signed certificate
Error: unable to verify the first certificate
Error: SELF_SIGNED_CERT_IN_CHAIN
Error: DEPTH_ZERO_SELF_SIGNED_CERT
NodeSslError: SSL Issue: consider using the 'Ignore SSL issues' option   ← n8n's wrapped version
```

Two paths to a fix; one is right and one is convenient.

### Right answer — `NODE_EXTRA_CA_CERTS`

Get a copy of the CA cert that signed your internal certs. For a self-signed cert, the CA *is* the cert itself.

```bash
# If the cert isn't already in PEM format, convert it
openssl x509 -in internal.crt -out internal.pem -outform PEM
```

Mount it into the container and point Node at it:

```yaml
services:
  n8n:
    environment:
      - NODE_EXTRA_CA_CERTS=/certs/internal-ca.pem
    volumes:
      - ./certs/internal-ca.pem:/certs/internal-ca.pem:ro
```

Restart n8n. Verify with a quick workflow:

```bash
docker compose exec n8n node -e \
  "require('https').get('https://your-internal-host/', r => console.log('OK', r.statusCode))"
```

If it prints `OK 200`, the cert is trusted.

### Convenient (and dangerous) — `NODE_TLS_REJECT_UNAUTHORIZED=0`

```yaml
environment:
  - NODE_TLS_REJECT_UNAUTHORIZED=0
```

Disables TLS verification globally for the n8n process. Works for the self-signed cert. Also disables verification for *every other* HTTPS call from n8n — including connections to public APIs over the internet. A man-in-the-middle on your network can now intercept all of n8n's outbound HTTPS.

**Acceptable** in:

- Local dev environments
- Throwaway demo containers
- Pre-production where you'll switch to `NODE_EXTRA_CA_CERTS` before going live

**Not acceptable** in:

- Anything labelled "prod"
- Anything carrying customer credentials, OAuth tokens, or PII
- Anything reachable from a network that isn't 100% under your control

### Per-node alternative — "Ignore SSL Issues" toggle

Each HTTP Request node has an *Options → Ignore SSL Issues* toggle. Same effect as `NODE_TLS_REJECT_UNAUTHORIZED=0`, but scoped to just that one node. Better than the global env var if you only have a couple of nodes hitting self-signed endpoints. Still: prefer `NODE_EXTRA_CA_CERTS` if the cert is stable, because then *no* toggle is needed and TLS verification protects you everywhere.

### Multiple internal CAs

`NODE_EXTRA_CA_CERTS` accepts a single file path, but the file can contain multiple certs concatenated:

```bash
cat ca-1.pem ca-2.pem ca-3.pem > /certs/internal-bundle.pem
```

Then `NODE_EXTRA_CA_CERTS=/certs/internal-bundle.pem`.

## Decision tree — when the customer's HTTP request still fails

A walk-through script for triaging tickets in this cluster:

1. *"What's the exact error message?"*
   - Contains `ECONNREFUSED 127.0.0.1` or `::1` → **Scenario 1**, the URL uses `localhost`
   - Contains `ENOTFOUND <hostname>` and the hostname is a real internal one → **Scenario 2**, set `HTTP_PROXY` / `HTTPS_PROXY`
   - Contains `ENOTFOUND host.docker.internal` → **Scenario 1**, on Linux, `extra_hosts` is missing
   - Contains `self signed certificate` / `unable to verify` / `SELF_SIGNED_CERT_IN_CHAIN` / n8n's *"SSL Issue"* → **Scenario 3**, set `NODE_EXTRA_CA_CERTS`
   - Contains `ETIMEDOUT` → either Scenario 2 (need proxy), or a firewall on the destination side. Ask whether the destination is reachable from the host with `curl -v <url>`
2. *"Can the host reach the URL?"* `curl -v <url>` from outside Docker. If the host can reach it, the bug is container-side. If the host can't reach it either, it's a network problem, not an n8n problem.
3. *"Are you on Docker Desktop or Linux Docker?"* Half the answers depend on this — see the per-platform tables in each scenario above.
4. *"Have you set proxy env vars anywhere else?"* Customers who've configured a system-wide proxy in their shell often don't realize Docker doesn't inherit it. Env vars must be set on the **n8n container's** environment, not the user's shell.

That sequence resolves the vast majority of tickets in this cluster in under 5 messages.
