# Repro 05 — Webhook delivery, idempotency & "double trigger"

> ⏳ **Planned.** See [`repros/01-webhook-url/`](../01-webhook-url/) for the pattern this folder will follow.

## What this will reproduce

Webhooks misfiring is the most "Slack-channel-on-fire" category. The repro will demonstrate:

- A workflow that triggers twice on a single inbound webhook (provider retry on 5xx)
- WhatsApp Cloud API webhook delays of minutes-to-hours
- Gmail Search re-firing after success
- Workflow not idempotent → duplicate side effects
- Test vs. Production webhook URL confusion

Each scenario ships with a small "fake provider" container that hammers the webhook with realistic retry semantics so you can watch the dedupe pattern (or lack thereof) in real time.

## Source threads

- [WhatsApp Cloud API webhooks delayed](https://community.n8n.io/t/whatsapp-cloud-api-webhooks-delayed-by-minutes-to-hours/265580) — 711 views
- [The "double trigger" problem (Slack)](https://community.n8n.io/t/the-n8n-double-trigger-problem-what-i-learned-building-a-slack-approval-flow/283332)
- [Workflow re-triggers Gmail Search after success](https://community.n8n.io/t/workflow-re-triggers-gmail-search-node-after-successful-completion/293235)
- [Idempotency & checkpointing issue](https://community.n8n.io/t/n8n-workflow-fails-to-resume-safely-after-partial-execution-idempotency-checkpointing-issue/293072)
- [Intermittent 403 from Google Apps Script](https://community.n8n.io/t/intermittent-403-forbidden-error-from-google-apps-script-to-webhook/276370)
- [Webhook production URL only working in test](https://community.n8n.io/t/webhook-production-url-only-working-in-test/80228)
