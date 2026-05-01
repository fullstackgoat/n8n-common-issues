# Repro 02 — OAuth 2.0 credential failures

> ⏳ **Planned.** See [`repros/01-webhook-url/`](../01-webhook-url/) for the pattern this folder will follow.

## What this will reproduce

OAuth 2.0 credential creation/refresh failures across self-hosted n8n. Specifically:

- Callback URL pointing to `localhost:5678` instead of the real domain
- Refresh-token expiry on Google / Microsoft OAuth credentials
- The `allowedDomains` Public-API contradiction
- Inoreader / Calendly / etc. OAuth that fails with `self-hosted n8n + ngrok`
- "Could not connect to your MCP server" SSE auth errors

## Source threads

- [OAuth callback URL is localhost:5678](https://community.n8n.io/t/oauth-callback-url-is-localhost-5678/2867) — 34.6k views
- [Google Calendar OAuth2 token expiring](https://community.n8n.io/t/google-calendar-oauth2-api-token-expiring-every-once-in-a-while/11336) — 9.5k views
- [Error 403 using OAuth Credential from Google](https://community.n8n.io/t/error-403-using-oauth-credential-from-google/3848) — 5.9k views
- [Public API "allowedDomains" errors](https://community.n8n.io/t/issue-with-updating-credentials-via-public-api-contradictory-alloweddomains-errors/293219)
- [Inoreader OAuth2 + ngrok](https://community.n8n.io/t/inoreader-oauth2-integration-fails-with-self-hosted-n8n-ngrok/253489)
- ["Could not connect to your MCP server" SSE](https://community.n8n.io/t/error-could-not-connect-to-your-mcp-server-when-integrating-external-tool-via-sse-in-ai-agent/100957) — 15.4k views
