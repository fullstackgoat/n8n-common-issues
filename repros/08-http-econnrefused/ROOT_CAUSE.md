# ROOT CAUSE — Repro 08

## TL;DR

n8n's HTTP Request node is doing exactly what you tell it. The three failures in this repro are all the same higher-order shape: **the n8n container's network and trust environment doesn't match what the workflow URL implies.** Three orthogonal pieces of that environment — DNS for host services, the proxy env vars, and Node.js' TLS trust store — have to be set on the n8n container itself, not inside the workflow.

## Layer 1 — `localhost` inside a container is the container itself

Every Linux container gets its own network namespace. The `lo` interface in there is private; `127.0.0.1` and `::1` are the container's own loopback. When n8n is running inside its container and a workflow does `GET http://localhost:11434/`, the request goes to port 11434 on the n8n container's `lo`. Nothing's listening. The kernel returns `ECONNREFUSED` — and on current Node.js (22+), the address shown in the error will usually be `::1:11434`, the IPv6 loopback, because Node now prefers IPv6 first when both families are available. (Older Node versions show `127.0.0.1:11434`. Same root cause.)

The user's mental model — "Ollama is running on `localhost:11434` on my Mac, n8n should be able to reach `localhost:11434`" — is reasonable everywhere except inside a container. The container has its own `localhost`.

### How to reach the host's network from inside a container

Three patterns, in increasing portability:

1. **`host.docker.internal`** — the canonical Docker DNS name for "the host's gateway address from a container's perspective." Built in on Docker Desktop (macOS, Windows). On Linux you must add it explicitly:
   ```yaml
   extra_hosts:
     - "host.docker.internal:host-gateway"
   ```
   Once added, `host.docker.internal:11434` from inside the container reaches the host's `127.0.0.1:11434` and (via the published port mapping) lands in `fake-ollama` or whatever else is listening on the host.

2. **The host's actual LAN IP** — `192.168.x.y` or whatever your machine has. Works always, but breaks on every DHCP renewal and isn't portable across machines.

3. **`network_mode: host`** — tells Docker to skip network isolation entirely and put the container directly on the host's network namespace. The container's `localhost` *becomes* the host's `localhost`. Powerful but throws away most of Docker's networking benefits and breaks Docker Compose's service-name DNS for sibling services. Generally a code smell.

The Ollama-on-host complaint is so common because the Ollama server (and every "run your own LLM locally" guide) tells you to use `localhost:11434`. That's correct guidance for a process on the host. It's wrong guidance for a process *inside a container on* the host. n8n surfaces the symptom; the wrong port assumption causes it.

## Layer 2 — Node.js' proxy semantics and `proxy-from-env`

When n8n's HTTP Request node fires, the request goes through the standard Node.js HTTP libraries. Those libraries respect a **per-process environment** for proxying:

- `HTTP_PROXY` — proxy URL for `http://` requests
- `HTTPS_PROXY` — proxy URL for `https://` requests (sent via HTTP `CONNECT` tunnels)
- `NO_PROXY` — comma-separated list of hostnames/IPs that should bypass the proxy

There are two subtleties:

### Subtlety A — Capitalization and which library reads which

Different HTTP libraries in the Node ecosystem read the env vars with different conventions:

- **`global-agent` / `proxy-agent`** — read both `HTTP_PROXY` and `http_proxy` (case-insensitive).
- **Native `fetch` / `undici`** — read `HTTP_PROXY` only when the agent is configured with `proxy-from-env`. If the library code doesn't opt in, the env vars are ignored.
- **`axios` and most `fetch`-based libs** — read `HTTPS_PROXY` for `https://` URLs but only when wrapped with the right agent.

n8n's HTTP Request node uses a configuration that respects the standard env vars. As long as `HTTP_PROXY` and `HTTPS_PROXY` are set on the n8n process (not just exported in the shell of whoever started Docker), outbound requests are proxied.

### Subtlety B — `NO_PROXY` matters

If `NO_PROXY` isn't set, every outbound request — including DNS-resolvable sibling services on the same Docker network — round-trips through the proxy. That's slow at best and can fail outright if the proxy refuses to relay to internal hostnames. The fixed stack sets:

```bash
NO_PROXY=localhost,127.0.0.1,n8n,fake-ollama,self-signed,self-signed-server,self-signed.local,host.docker.internal
```

…so requests to those names go direct, and only requests to `internal-api` (which is genuinely unreachable from n8n's network) get forwarded to Squid. In real corporate setups, `NO_PROXY` is usually `*.internal.example.com,10.0.0.0/8,localhost` or similar — exclude everything you can route to directly, proxy everything else.

### Why the bug is silent

If `HTTP_PROXY` is unset, n8n's HTTP Request just goes direct. There's no warning, no banner. The user can't tell from the editor that they're missing a proxy config until the request fails — and even then, the error (`ENOTFOUND` / `ETIMEDOUT`) doesn't mention that a proxy might have been needed. The forum thread *"HTTP Request via proxy ECONNREFUSED 127.0.0.1:80"* is exactly this confusion: the user *did* try to set a proxy, but did it inside the workflow's request options instead of as an env var on the container, so n8n tried to literally connect to `127.0.0.1:80` looking for a local proxy that doesn't exist.

## Layer 3 — Node.js' TLS trust store

When Node.js makes an HTTPS request, it validates the server's certificate chain against an in-process trust store. By default the store is a hardcoded list of public CAs that Node ships with (Mozilla's bundle, baked into the binary). Any cert not signed by one of those — including every self-signed cert and every cert from a private internal CA — fails verification with one of:

```
Error: self signed certificate
Error: unable to verify the first certificate
Error: SELF_SIGNED_CERT_IN_CHAIN
Error: DEPTH_ZERO_SELF_SIGNED_CERT
```

(The exact wording is a function of which Node version and how deep into the chain the failure was.)

There are two env vars that change this behaviour:

### `NODE_EXTRA_CA_CERTS` — the right answer

```yaml
- NODE_EXTRA_CA_CERTS=/certs/internal-ca.crt
```

A path to a PEM file containing one or more CA certs. Node reads it once at process startup and adds those CAs to the in-process trust store. From that point on, certs signed by them verify successfully — and **TLS verification stays on for everything else**. This is the Right Answer for any environment with internal CAs.

Limitations:

- Read once at startup. If the file changes, restart the process.
- Per-process. If you have multiple Node processes (n8n + workers), set it on each.

### `NODE_TLS_REJECT_UNAUTHORIZED=0` — the wrong answer (in production)

```yaml
- NODE_TLS_REJECT_UNAUTHORIZED=0   # don't ship this to prod
```

Disables TLS verification for **every** outbound HTTPS call from the Node process. Solves the immediate problem; introduces a new one — your n8n now happily talks to any TLS endpoint regardless of cert validity, which is fine in dev and unacceptable in prod (a man-in-the-middle in your network can intercept any HTTPS call from n8n).

n8n has been deliberate about not silently disabling TLS. If you set this env var, Node prints a warning at startup. Pay attention to that warning — and use `NODE_EXTRA_CA_CERTS` instead the moment you have a stable cert to point at.

### What to do per-environment

| Environment | Recommendation |
|---|---|
| Local dev with self-signed cert | `NODE_EXTRA_CA_CERTS` if the cert is stable; `NODE_TLS_REJECT_UNAUTHORIZED=0` is acceptable for throwaway demos |
| Internal production with corporate CA | Always `NODE_EXTRA_CA_CERTS` pointing at the corporate CA bundle |
| Public internet | Use a real cert from Let's Encrypt or any public CA — neither env var should be set |

## Why this is the #1 self-hosting gotcha after persistence

Three structural reasons:

1. **The defaults are friendly to a single laptop and hostile to containers.** `localhost`, the system trust store, no proxy — all reasonable defaults for a laptop process. All wrong assumptions for a process inside a container.
2. **The error messages don't suggest the right fix.** `ECONNREFUSED` on `127.0.0.1` doesn't say *"are you sure you meant the container's loopback?"*. `unable to verify the first certificate` doesn't say *"set NODE_EXTRA_CA_CERTS to your CA bundle."* Every error needs a layer of translation between what Node says and what the user needs to do.
3. **There's no central place in n8n to configure these.** Each setting lives on the container's environment, not in n8n's settings UI. A user can't introspect their proxy configuration through the editor — they have to know to look at the container env vars, which most don't.

## Forum citations

- [HTTP Request via proxy `ECONNREFUSED 127.0.0.1:80`](https://community.n8n.io/t/http-request-via-proxy-connect-econnrefused-127-0-0-1-80/8469) — 24.5k views — scenario 2; user set proxy in workflow request options instead of container env
- [Unable to connect n8n to local Ollama instance — `ECONNREFUSED`](https://community.n8n.io/t/unable-to-connect-n8n-to-local-ollama-instance-econnrefused-error/90508) — 10.9k views — scenario 1; the canonical version of the `localhost` mistake
- [Self-signed certificate error when setting MS SQL Server connection](https://community.n8n.io/t/self-signed-certificate-error-when-setting-ms-sql-server-connection/18863) — scenario 3 in a different node, same root cause
- [AWS Bedrock invoke using HTTP node not working](https://community.n8n.io/t/aws-bedrock-invoke-using-http-node-not-working/292675) — adjacent — request signing failures, will be covered separately if Project 2 demands it

## Upstream issue

The single highest-leverage product fix would be a **HTTP Request node "diagnose"** action: a button in the node's parameters panel that, given a URL, runs four probes and reports which one(s) failed:

1. DNS lookup of the hostname from inside the n8n container
2. TCP connect to the resolved address
3. TLS handshake (if `https://`) with verification
4. TLS handshake without verification (to distinguish "unreachable" from "untrusted cert")

The output would say things like *"DNS lookup failed — is `host.docker.internal` configured? See: extra_hosts"* or *"TLS handshake failed verification but succeeded without — server uses a self-signed or internal-CA cert. See: NODE_EXTRA_CA_CERTS."*

Today, the user has to triangulate this themselves from a generic Node.js error message. A diagnose button would deflect a measurable fraction of the tickets in this cluster. The infrastructure for it already exists in n8n's HTTP node code; the diagnostic mode is the missing UI affordance.
