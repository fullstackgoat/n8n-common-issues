# `n8n-debug-cli`

> ⏳ **Planned.** A small Node.js CLI that scales support work — turning common debugging tasks into one-line commands.

## Planned commands

```bash
n8n-debug repro <cluster>           # boots a docker-compose repro from this monorepo
n8n-debug logs  <execution.json>    # parses an execution export, surfaces error/stack/timing outliers
n8n-debug webhook <url> [--secret]  # sends test payloads with HMAC, header inspection
n8n-debug oauth <url>               # decodes JWT, traces redirect chain, checks callback URL
n8n-debug net   <host>              # one-shot DNS/TLS/HTTP/proxy diagnostic
```

## Roadmap

- [ ] `logs` (highest daily-use value)
- [ ] `webhook`
- [ ] `repro` (wraps `docker compose up` against repros in this monorepo)
- [ ] `oauth`
- [ ] `net`
- [ ] Publish to npm as `n8n-debug-cli`
- [ ] GitHub Actions CI: lint, test, npm publish on tag

## Issue coverage

When complete, this CLI addresses Issues 01, 02, 05, 07, 08, 09 from the [Top 10 Issues report](../../docs/top-10-issues.html).

## Tech

Plain Node.js (no framework), `commander` for argument parsing, `undici` for HTTP. Stays under ~500 LOC; vendor only what's strictly needed.
