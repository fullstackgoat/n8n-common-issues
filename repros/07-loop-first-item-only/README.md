# Repro 07 — Loops, iteration & data-shape errors

The most teachable repro in the set. Almost always user-error; almost always KB-able in 90 seconds. Two related failure modes:

1. *"My loop only processes the first item"* — caused by Code-node misuse, not by n8n's iteration model
2. *"Input values have N keys, you must specify an input key or pass only 1 key as input"* — caused by multi-key items feeding a node that wants exactly one

## Symptoms (any combination)

- A workflow with N input items only produces 1 downstream effect
- Wrapping the work in a "Loop Over Items" node doesn't change anything
- A Code node that "looks right" silently drops items 2 through N
- The error message *"Input values have N keys, you must specify an input key or pass only 1 key as input"* when calling a sub-workflow, an AI Agent tool, or the Aggregate node
- "Undefined" data showing up after a loop

## What this repro proves

That every symptom above stems from one of two root causes:

1. **A Code node that hard-codes `items[0]`** (or `$input.first()`, or `$json` in *Run Once for All Items* mode), masquerading as a loop bug
2. **Multi-key items feeding a node that needs single-key input** (Sub-Workflow Trigger, AI Agent tools, Aggregate)

…and that fixing the Code node's mode or its iteration pattern, OR reshaping items with a single Set node, eliminates every symptom.

This is workflow-only — there's no service to misconfigure, no docker-networking to debug. The bugs all live in node configuration and code.

## Files

| File | Purpose |
|---|---|
| [`workflows/broken-loop.json`](workflows/broken-loop.json) | The "only first item" bug — import & run, watch 5 items collapse to 1 |
| [`workflows/fixed-loop.json`](workflows/fixed-loop.json) | Side-by-side fixes — Pattern A (`items.map()`), Pattern B (*Run Once for Each Item* mode) |
| [`workflows/three-keys-error.json`](workflows/three-keys-error.json) | Reproduces *"Input values have N keys"* exactly, with the fix explained inline |
| [`docker-compose.yml`](docker-compose.yml) | Vanilla n8n — optional, only if running locally instead of in n8n cloud |
| [`.env.example`](.env.example) | Env vars for the local container |
| [`REPRO.md`](REPRO.md) | Step-by-step reproduction with observed vs expected output |
| [`ROOT_CAUSE.md`](ROOT_CAUSE.md) | n8n's items model and fan-out semantics, with diagram |
| [`WORKAROUND.md`](WORKAROUND.md) | Copy-pasteable fixes you can paste into a ticket reply |

## Quick start

### Option A — n8n cloud (zero setup)

1. Open your n8n cloud workspace
2. Create a new workflow → top-right `⋯` menu → **Import from File**
3. Pick any of the three JSONs in [`workflows/`](workflows/)
4. Click **Test workflow** at the bottom of the canvas

The workflows are self-contained — they generate their own input data, so no credentials or external APIs are required.

### Option B — Local Docker

```bash
cd repros/07-loop-first-item-only
cp .env.example .env
docker compose up -d
```

Open <http://localhost:5678>, create an owner account, then import the workflows the same way as Option A.

## Recommended troubleshooting flow

For maximum learning, run the workflows in this order:

1. **`broken-loop.json`** — confirm 5 → 1 collapse on `Process Lead (BUG)`
2. **Insert a Loop Over Items node before the buggy Code node** *(optional but recommended)* — see that wrapping it doesn't help
3. **`fixed-loop.json`** — both branches (Pattern A and Pattern B) produce 5 items
4. **Edit the Pattern A node back to `items[0]`** — watch that branch break while Pattern B stays green
5. **`three-keys-error.json`** — see the *"Input values have N keys"* error, then apply the inline Set-node fix

That sequence takes ~10 minutes and covers every failure mode in this cluster.

## Linked KB article

[`kb/07-loops.md`](../../kb/07-loops.md) — the customer-facing fix you can paste into a ticket reply.

## Source threads

- [Loop Node only processes first item](https://community.n8n.io/t/loop-node-only-processes-the-first-item-how-to-fix/56727) — 6.8k views
- ["Input values have 3 keys"](https://community.n8n.io/t/input-values-have-3-keys-you-must-specify-an-input-key-or-pass-only-1-key-as-input/74622) — 7.3k views
- [Have your say: improve the Loop Over Items node](https://community.n8n.io/t/have-your-say-help-us-improve-the-loop-over-items-node/92570) — n8n staff thread on this exact pain
