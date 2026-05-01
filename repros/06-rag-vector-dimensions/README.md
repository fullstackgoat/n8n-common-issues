# Repro 06 — AI Agent / RAG production failures

> ⏳ **Planned.** See [`repros/01-webhook-url/`](../01-webhook-url/) for the pattern this folder will follow.

## What this will reproduce

The fastest-growing ticket category. Repro stack: n8n + Qdrant + Supabase pgvector + a known-good corpus.

Failures demonstrated:

- Vector dimension mismatch (e.g. embedding model emits 1536, vector store column is 192)
- Embeddings node silently not calling OpenAI during inserts
- Supabase Vector Store returning empty arrays despite SQL matches
- "Received tool input did not match expected schema" on AI Agent
- MCP/SSE transport blocked by reverse-proxy buffering
- AI Agent latency at 16s+ in production

Includes a tiny eval harness (RAGAS-style precision@k) so customers can confirm green/red in under 5 minutes.

## Source threads

- [AI Agent too slow for production RAG (16s+)](https://community.n8n.io/t/is-the-ai-agent-node-too-slow-for-production-rag-seeing-16s-response-times/287840)
- [Embeddings node not calling OpenAI](https://community.n8n.io/t/n8n-vector-store-retrieves-correctly-but-embeddings-openai-node-never-seems-to-call-the-openai-api-during-fresh-document-inserts/279480)
- [Vector nodes store wrong dimension](https://community.n8n.io/t/vector-nodes-store-wrong-dimension-always-192/293231)
- [Supabase Vector Store returns empty](https://community.n8n.io/t/supabase-vector-store-node-returns-empty-output-no-error-while-sql-query-returns-matches/267685)
- [Received tool input did not match schema](https://community.n8n.io/t/received-tool-input-did-not-match-expected-schema/75832) — 14.1k views
- [MCP/SSE connection error](https://community.n8n.io/t/error-could-not-connect-to-your-mcp-server-when-integrating-external-tool-via-sse-in-ai-agent/100957) — 15.4k views
