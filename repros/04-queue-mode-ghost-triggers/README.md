# Repro 04 — Queue mode ghost triggers & worker race conditions

> ⏳ **Planned.** See [`repros/01-webhook-url/`](../01-webhook-url/) for the pattern this folder will follow.

## What this will reproduce

The hardest cluster — and the one IC3 hires are expected to own.

The repro will boot n8n in queue mode with main + 2 workers + Redis + Postgres, then trigger:

- Cron triggers firing N times because they registered on every worker instead of only main
- "Worker failed to find data for execution" race condition (v2.x)
- UI failing to render in queue mode
- Workflow `Postgres` node failing because the queue worker doesn't have the credential mounted

## Source threads

- [Cron ghost triggers in queue mode](https://community.n8n.io/t/cron-trigger-executing-multiple-times-after-updates-due-to-ghost-triggers-in-queue-mode-with-multiple-workers/244687)
- [v2.2.4 Worker race condition](https://community.n8n.io/t/v2-2-4-worker-failed-to-find-data-for-execution-race-condition-in-queue-mode-not-present-in-v1-x/246815)
- [UI won't render in queue mode](https://community.n8n.io/t/ui-wont-render-in-queue-mode/192160)
- [Redis broker issues on Cloud Run](https://community.n8n.io/t/redis-and-broker-connection-issues-in-n8n-queue-mode-cloud-run-deployment/176093)
- [Postgres node fails in queue mode](https://community.n8n.io/t/postgres-node-fails-in-queue-mode-cannot-read-properties-of-undefined-reading-execute/111744)
- [High RAM in queue mode](https://community.n8n.io/t/high-usage-of-ram-in-queue-mode-no-reset/63141)
