# Loop only processes the first item, or "Input values have N keys" error

If any of these sound familiar:

- Your workflow has 5 input items but only 1 downstream effect happens
- You wrapped the work in a "Loop Over Items" node and nothing changed
- A Code node "looks right" but only the first item makes it through
- You're seeing the error: *"Input values have N keys, you must specify an input key or pass only 1 key as input"*
- "Undefined" data appears in nodes after a loop completes

…you're hitting the most common workflow-design issue in n8n. Both flavours of this bug are usually fixable in two minutes.

## Confirm it's this issue

In your workflow:

1. Click the upstream node (the one producing multiple items) → it should show **N** items in its output panel
2. Click the next node → if it shows fewer items than expected, that's the break point
3. Open that node — if it's a **Code node**, the bug is almost certainly inside the code

OR — if you have an explicit error message *"Input values have N keys, you must specify an input key or pass only 1 key as input"*, jump to *Fix B* below.

## Why it happens

n8n already loops. Every node downstream of a multi-item source executes once per input item — automatically. There is no implicit "current item" across the whole workflow; each node sees its own input array, and each node processes that array end-to-end.

Two ways users break this:

1. **A Code node that hard-codes `items[0]`** — looks like it's processing "the current item," is actually picking the first one and discarding the rest
2. **Multi-key items feeding a node that needs single-key input** — Sub-Workflow Trigger, AI Agent tools, and the Aggregate node all check the shape of incoming items and refuse multi-key input with a specific error

## Fix

### Fix A — Code node only processing the first item

Look in your Code node for any of these patterns:

```js
const lead = items[0].json;
const lead = $input.first().json;
const lead = $json;     // in 'Run Once for All Items' mode — same trap
return [{ json: lead }];
```

Replace with one of:

**Pattern 1 — `items.map()` in default mode**

```js
return $input.all().map(item => ({
  json: {
    ...item.json,
    // your transformations here
  }
}));
```

**Pattern 2 — Switch the Code node Mode dropdown to *Run Once for Each Item***

```js
const lead = $input.item.json;
return {
  json: {
    ...lead,
    // your transformations here
  }
};
```

Returns a single object, not an array. n8n handles iteration for you.

You don't need a Loop Over Items node. n8n already iterates downstream of any multi-item source.

### Fix B — "Input values have N keys" error

Drop a **Set** (Edit Fields) node before the offending receiver. Wrap your multi-field object under a single key:

```
Field name:  data
Field type:  Object
Field value: ={{ $json }}
```

Toggle **Include Other Input Fields:** off, so only `data` ends up in the output.

Each item now has one top-level key (`data`) whose value is the original multi-field object. The Sub-Workflow Trigger / AI Agent / Aggregate node will accept it.

Alternatively, if the receiving node has a *Prompt* / *Input Field* parameter (e.g. AI Agent), point it at one of your existing fields directly instead of reshaping.

## Prevent it next time

- When writing a Code node, default to `$input.all().map(...)` — never `items[0]`
- Use *Run Once for Each Item* mode when you want clean per-item logic with a strict 1-to-1 mapping
- Don't add a Loop Over Items node "to make it iterate" — n8n already does. Use it only for genuine batching (paginated API calls, rate-limited APIs)
- If a downstream node complains about multi-key input, reshape with a Set node before the node — don't fight the node
- Check non-Code nodes for an **Execute Once** toggle when they unexpectedly emit a single item

## Related forum threads

- [Loop Node only processes first item](https://community.n8n.io/t/loop-node-only-processes-the-first-item-how-to-fix/56727) — 6.8k views
- ["Input values have 3 keys"](https://community.n8n.io/t/input-values-have-3-keys-you-must-specify-an-input-key-or-pass-only-1-key-as-input/74622) — 7.3k views
- [Have your say: improve the Loop Over Items node](https://community.n8n.io/t/have-your-say-help-us-improve-the-loop-over-items-node/92570) — n8n staff thread

## Reproduction

Want to see all three failure modes side-by-side? The full reproduction is at [`repros/07-loop-first-item-only/`](../repros/07-loop-first-item-only/) — three importable workflow JSONs that produce each bug, plus the fixed counterparts.
