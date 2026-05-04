# REPRO — Repro 07

This repro is workflow-only — no Docker stack is required to reproduce the bugs themselves. You can import the three workflow JSONs into any n8n instance (including n8n cloud) and run them with the **Test workflow** button. Each workflow generates its own input data, so no credentials or external APIs are needed.

If you'd rather run them in a local container, an optional `docker-compose.yml` is provided.

## Setup (optional — only if running locally)

```bash
cd repros/07-loop-first-item-only
cp .env.example .env
docker compose up -d
```

Wait ~10 seconds for n8n to boot:

```bash
docker compose logs -f n8n | grep "Editor is now accessible"
# Press Ctrl-C once you see it
```

Open <http://localhost:5678>, create an owner account, then proceed.

## Symptom 1 — Loop only processes the first item

1. Import [`workflows/broken-loop.json`](workflows/broken-loop.json) into your n8n instance (top-right `⋯` menu → **Import from File**).
2. Click **Test workflow** at the bottom of the canvas.
3. Click each node in turn and inspect the output panel.

**Observed:**

| Node | Items out |
|---|---|
| `Generate 5 Leads` | **5** |
| `Process Lead (BUG)` | **1** ← bug surfaces here |
| `Mark as Synced` | **1** |

**Expected:** All 5 items should make it through; all 5 leads should be marked as synced.

**Why we see it:** The Code node in `Process Lead (BUG)` does `const lead = items[0].json` — it hard-codes the first item instead of mapping over all of them. n8n is iterating fine; the Code node is the broken link, and 4 leads are dropped silently.

## Symptom 2 — Wrapping in "Loop Over Items" doesn't fix it

1. Edit the broken workflow: between `Generate 5 Leads` and `Process Lead (BUG)`, insert a **Loop Over Items** (SplitInBatches) node with default settings. Leave the `done` output disconnected.
2. Run again.

**Observed:** The Loop Over Items node fires 5 iterations (visible in its iteration counter). Final output of `Mark as Synced` is still **1 item**.

**Expected:** Wrapping in a loop should make the workflow process all 5 items.

**Why we see it:** Loop Over Items is iterating correctly. Each iteration runs the same buggy Code node, which still returns `items[0]` of its 1-item batch. The bug is two lines inside the Code node — n8n's iteration model is not the problem.

## Symptom 3 — All five items make it through after the fix

1. Remove the Loop Over Items node from the previous step.
2. Import [`workflows/fixed-loop.json`](workflows/fixed-loop.json).
3. Click **Test workflow**.

**Observed:** Both branches of the fixed workflow output 5 items at every node:

| Branch | Pattern | Mark as Synced output |
|---|---|---|
| Top | `items.map()` in *Run Once for All Items* mode | **5** |
| Bottom | *Run Once for Each Item* mode | **5** |

**Expected:** Both fix patterns work; both produce 5 items.

**Why it works:** Pattern A keeps the Code node in default mode but explicitly iterates with `$input.all().map(...)`. Pattern B switches the node's *Mode* dropdown to *Run Once for Each Item*, which makes n8n run the code body once per input item; `$input.item.json` then refers to the current item.

## Symptom 4 — "Input values have N keys" error from a sub-workflow / AI Agent

1. Import [`workflows/three-keys-error.json`](workflows/three-keys-error.json).
2. Click **Test workflow**.

**Observed:** The execution halts at `Single-Key Validator` with the exact error:

```
Input values have 3 keys, you must specify an input key or pass only 1 key as input
```

**Expected:** A workflow that produces multi-field items should be able to feed any downstream node.

**Why we see it:** Several real n8n nodes — Execute Sub-Workflow Trigger in single-key input mode, AI Agent / Tools (LangChain wrappers expect a single `input` field), Aggregate in *Aggregate Individual Fields* mode without a field specified — check the shape of incoming items and refuse multi-key input. The validator Code node in this workflow simulates that internal check, so the error is reproducible without wiring up a sub-workflow.

## Apply the fix for Symptom 4

1. In the broken workflow, right-click `Single-Key Validator` → **Insert before** → add an **Edit Fields (Set)** node.
2. Configure it (Manual Mapping):
   - **Field name:** `data`
   - **Field type:** Object
   - **Field value:** `={{ $json }}`
   - **Include Other Input Fields:** off
3. Run again.

**Expected after fix:** Each item now has one top-level key (`data`) whose value is the original `{ id, name, email }` object. The validator passes.

## Teardown

```bash
docker compose down -v
```
