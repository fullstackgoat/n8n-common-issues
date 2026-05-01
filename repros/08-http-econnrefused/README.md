# Repro 08 — HTTP Request & API debugging (proxies, ECONNREFUSED, custom auth)

> ⏳ **Planned.** See [`repros/01-webhook-url/`](../01-webhook-url/) for the pattern this folder will follow.

## What this will reproduce

The Senior Support Engineer's bread-and-butter cluster. Stacks shipped:

- n8n + a fake "corporate proxy" container, demonstrating `HTTP_PROXY`/`HTTPS_PROXY` env-var setup
- n8n container trying (and failing) to reach `localhost:11434` for Ollama → fix is `host.docker.internal`
- n8n calling AWS Bedrock with hand-rolled SigV4 vs. via the AWS node
- n8n + a self-signed-cert MS SQL Server, demonstrating `NODE_TLS_REJECT_UNAUTHORIZED` and proper CA bundling
- HMAC signature debugging (Bybit-style) with a tiny verifier you can copy-paste

## Source threads

- [HTTP Request via Proxy ECONNREFUSED](https://community.n8n.io/t/http-request-via-proxy-connect-econnrefused-127-0-0-1-80/8469) — 24.5k views
- [Unable to connect to local Ollama](https://community.n8n.io/t/unable-to-connect-n8n-to-local-ollama-instance-econnrefused-error/90508) — 10.9k views
- [AWS Bedrock invoke not working](https://community.n8n.io/t/aws-bedrock-invoke-using-http-node-not-working/292675)
- [Self-signed cert error MS SQL Server](https://community.n8n.io/t/self-signed-certificate-error-when-setting-ms-sql-server-connection/18863)
- [Snowflake key-pair authentication](https://community.n8n.io/t/snowflake-key-pair-authentification/84720)
