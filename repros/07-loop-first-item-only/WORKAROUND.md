# WORKAROUND — Repro 07

A copy-pasteable fix you can apply to a customer's workflow today, or paste into a ticket reply.

## Symptom 1 — "My loop is only processing the first item"

The bug is almost certainly inside a Code node, not in n8n's iteration model. Look for any of these patterns in the customer's Code:

```js
// Patterns that break — only ever return the first item
const lead = items[0].json;
const lead = $input.first().json;
const lead = $json;        // (in 'Run Once for All Items' mode — same trap)
return [{ json: lead }];
```

### Fix A — Stay in *Run Once for All Items*, use `items.map()`

```js
return $input.all().map(item => ({
  json: {
    ...item.json,
    // ...your transformations here
  }
}));
```

### Fix B — Switch to *Run Once for Each Item* mode

In the Code node parameters panel, change the **Mode** dropdown to *Run Once for Each Item*. Then:

```js
const lead = $input.item.json;
return {
  json: {
    ...lead,
    // ...your transformations here
  }
};
```

Returns a single object, not an array. n8n handles the iteration for you.

### When to use which

- **Pattern A** when you're transforming, deduplicating, batching, or fanning items in or out (1 → N or N → M mappings).
- **Pattern B** when you genuinely want clean per-item logic with a strict 1-to-1 mapping. Easier to read; slightly slower at high item counts.

## Symptom 2 — "Wrapping in Loop Over Items doesn't help"

It won't, because the bug is inside the Code node — the loop is doing its job, the code is not. The verbatim ticket reply that lands well:

> **Loop Over Items isn't the problem.** n8n is already iterating — every node downstream of a multi-item source executes once per item automatically. Your Code node hard-codes `items[0]`, which means each iteration of the loop is still only returning the first item of its batch. Fix the Code node (see *Fix A* or *Fix B* below) and you can remove the Loop Over Items node entirely.

## Symptom 3 — "Input values have N keys, you must specify an input key or pass only 1 key as input"

The receiving node expects items with exactly one JSON key, and the items have multiple. Common culprits:

- **Execute Sub-Workflow Trigger** in single-key input mode
- **AI Agent / Tools** nodes (LangChain wrappers expect a single `input` field)
- **Aggregate** node in *Aggregate Individual Fields* mode without a field specified

### Fix — Reshape input to a single key

Drop a **Set** (Edit Fields) node before the receiving node. Use *Manual Mapping* mode and create one field — typically named `data`, `input`, or whatever the downstream node expects:

```
Field name:  data
Field type:  Object
Field value: ={{ $json }}
```

Toggle **Include Other Input Fields:** off, so only the `data` key ends up in the output.

Each item now looks like:

```json
{ "data": { "id": 1, "name": "Acme Corp", "email": "founder@acme.test" } }
```

One top-level key. The receiving node accepts it.

### Alternative — Specify the input key on the receiving node

Several of the offending nodes (notably AI Agent) have a *Prompt* / *Input Field* parameter where you can name the specific JSON field they should use. If your data already has the key the node wants (e.g. `text`, `query`, `prompt`), just point the node at it instead of reshaping.

## Quick reference — Code node modes

### *Run Once for All Items* (default)

| | Value |
|---|---|
| `$input.all()` | `Item[]` — every input item |
| `$json` | `items[0].json` — **the first item, not "the current item"** |
| Must return | `Item[]` — an array |

### *Run Once for Each Item*

| | Value |
|---|---|
| `$input.item` | the current `Item` |
| `$json` | `$input.item.json` — really does mean "the current item" here |
| Must return | a single `Item` (not an array) |

The single most common confusion: assuming `$json` means "the current item" in either mode. It does in *Run Once for Each Item* mode. It does **not** in *Run Once for All Items* mode.

## Per-node notes

### Code node

Default mode is *Run Once for All Items*. If a customer wants per-item logic and you can't easily talk them through `items.map()`, switch their mode and rewrite the body — usually a smaller diff.

### HTTP Request, Set, Function

Each of these has an **Execute Once** toggle in node settings. If on, the node runs exactly once with the first input item, regardless of how many came in. Verify this is off when a non-Code node is dropping items.

### Loop Over Items / SplitInBatches

Use it for **batching** — processing items in groups of N (e.g. paginated API calls, rate-limited APIs). Not needed to "make a workflow iterate." If you do use it, remember:

- Connect the loop output (output 0) back into the node after the inner work
- Wire the `done` output (output 1) to whatever should run after all batches finish
- Don't increase batch size to "speed it up" — the value is in the rate-limit shape, not throughput

### AI Agent / Tools

If the customer hits *"Input values have N keys"*: either reshape with Set (above), or set the *Prompt* parameter to `={{ $json.fieldName }}` pointing at the specific field the agent should use as input.

### Aggregate

If you're in *Aggregate Individual Fields* mode, populate **Fields to Aggregate** with at least one field name. If you actually want to aggregate everything into one big array per group, switch the mode to *Aggregate All Item Data*.

## Walk-through script for a stuck customer

When a customer says "my loop isn't working" and you're triaging:

1. *"Click the upstream node — the one producing multiple items. How many items does it show in the output panel?"*
2. *"Click the next node. How many items in its output?"*
3. *"If those numbers don't match, that's where the bug lives."*
4. *"Open that node — is it a Code node? If yes, can you paste the code?"*
5. *"Look for `items[0]`, `$input.first()`, or `$json` in *Run Once for All Items* mode — those are the three patterns that drop items."*
6. *"If it's not a Code node, check the node settings panel for an **Execute Once** toggle."*
7. *"If you're seeing 'Input values have N keys', it's a different bug in the same family — reshape the input with a Set node, or point the receiving node at a specific field."*

That sequence resolves the vast majority of tickets in this cluster in under 10 messages.
