# Repro 10 — Provider node regressions after upgrade

> ⏳ **Planned.** See [`repros/01-webhook-url/`](../01-webhook-url/) for the pattern this folder will follow.

## What this will reproduce

The "it worked yesterday" cluster — every n8n release inevitably breaks at least one provider integration. Repro pattern: pin n8n to two adjacent versions and run the same workflow against both, capturing the diff.

Specific cases captured at scrape time:

- Snowflake Insert broken after 2.17.5
- Salesforce Case creation suddenly requires Type Name
- Gemini 3.0 tools "missing thought_signature"
- Official node disappearing from search results
- WhatsApp node behavior change after permanent API key
- Anthropic credential validation failing in n8n while working via curl

## Source threads

- [Snowflake Insert broken after 2.17.5](https://community.n8n.io/t/snowflake-insert-node-stopped-working-after-update-to-2-17-5/290608)
- [Can't create Salesforce Case without Type Name](https://community.n8n.io/t/cant-create-salesforce-case-without-a-type-name-value/293182)
- [Gemini 3.0 tools "missing thought_signature"](https://community.n8n.io/t/issue-with-gemini-3-0-gemini-3-pro-preview-tools-function-call-is-missing-a-thought-signature/223824)
- [Official node not showing in search results](https://community.n8n.io/t/bug-with-an-official-node-not-showing-up-in-n8n-templates-and-search-results-please-help/251467)
- [WhatsApp node after permanent API key](https://community.n8n.io/t/issue-with-whatsapp-node-after-using-permanent-api-key/78504)
- [Anthropic credential validation fails (curl works)](https://community.n8n.io/t/anthropic-credential-validation-fails-with-resource-not-found-but-api-key-works-via-curl/271100)
