# Repro 09 — Performance, memory & large-data crashes

> ⏳ **Planned.** See [`repros/01-webhook-url/`](../01-webhook-url/) for the pattern this folder will follow.

## What this will reproduce

The "is n8n down or is my workflow slow?" cluster.

Repro stack: n8n with a deliberately tight memory limit (e.g. 512 MB) + a Postgres seeded with 50k rows + a workflow that loads them all into a single item. Watch the OOM kill happen, then apply the fix (chunked SELECT + binary-data filesystem mode).

Also includes:

- A workflow with synchronous chained LLM calls (slow) vs. parallel branches (fast)
- A queue-mode stack with worker concurrency tuned past container memory budget

## Source threads

- [Workflow crashes on 40k SQL records — OOM](https://community.n8n.io/t/workflow-crashes-when-processing-large-sql-results-40k-records-container-memory-overflow/216917)
- [Why is AI Agent extremely slow](https://community.n8n.io/t/why-is-the-ai-agent-in-n8n-is-extremely-slow-when-processing-data/73442) — 7.2k views
- [Qdrant data loader slowing on large dataset](https://community.n8n.io/t/uploading-a-large-dataset-to-qdrant-data-loader-slowing-down/166481)
- [Long time until job is processed in queue mode](https://community.n8n.io/t/long-time-until-job-is-processed-on-queue-mode/112679)
- [Queue mode webhook concurrency testing](https://community.n8n.io/t/queue-mode-webhook-processing-concurrency-testing/66942)
