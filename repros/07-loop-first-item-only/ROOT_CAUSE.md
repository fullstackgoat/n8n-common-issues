# ROOT CAUSE — Repro 07

## TL;DR

n8n already loops. Every node downstream of a multi-item source executes once per input item, automatically. Bugs that look like *"the loop isn't working"* are almost always inside a Code node that hard-codes `items[0]`, or a node toggle (*Execute Once*) that explicitly bypasses iteration. A separate but related class of bug — *"Input values have N keys"* — has the opposite shape: items with too many JSON keys feeding a node that requires exactly one.

## Layer 1 — n8n's items model

In n8n, every node receives a list of items as input and emits a list of items as output. There is no implicit single "current item" across the whole workflow — each node's perspective is its own input array.

```
Trigger ─→ [item₀, item₁, item₂, item₃, item₄]
              │   │   │   │   │
              ▼   ▼   ▼   ▼   ▼
            Node A     (executes 5×, produces 5 output items)
              │   │   │   │   │
              ▼   ▼   ▼   ▼   ▼
            Node B     (executes 5×, produces 5 output items)
              │   │   │   │   │
              ▼   ▼   ▼   ▼   ▼
            Node C     (...)
```

The fan-out is automatic. You don't need a `Loop Over Items` node to "make it loop" — the loop is structural to n8n's execution model. The Loop Over Items / SplitInBatches node exists for **batching** (process N at a time, e.g. paginated API calls or rate-limited services), not for "making things iterate."

## Layer 2 — Where users actually break this

### 2a. Code node hard-codes `items[0]`

```js
const lead = items[0].json;
return [{ json: lead }];
```

This compiles, this runs, but only the first item is ever processed. The other N-1 items are discarded silently — no error, no warning, just missing data downstream. This is the modal n8n forum complaint, and it's what [`workflows/broken-loop.json`](workflows/broken-loop.json) demonstrates.

The fix is one of:

- `items.map(...)` in default *Run Once for All Items* mode
- Switch the node's **Mode** dropdown to *Run Once for Each Item* and use `$input.item.json`

Both patterns are shown side-by-side in [`workflows/fixed-loop.json`](workflows/fixed-loop.json).

### 2b. Misuse of `$json` inside *Run Once for All Items* mode

In *Run Once for All Items* mode, `$json` is shorthand for `items[0].json` — the **first** item, not "the current item." Many users assume `$json` is per-item-aware. It isn't, in that mode. Same trap as 2a, with a different syntactic disguise.

In *Run Once for Each Item* mode, `$json` does mean the current item (because the code body runs per item). Whether `$json` is "the current item" or "the first item" depends entirely on the node's mode — which is exactly the kind of context-sensitive surprise that fuels the forum complaints.

### 2c. *Execute Once* toggle on Set / HTTP Request / Function nodes

Some nodes have an **Execute Once** toggle in their settings. When on, the node runs exactly once with the first input item and emits one output item, regardless of how many items came in. Functionally equivalent to 2a, but caused by a UI toggle instead of code. Check this when a non-Code node is the break point.

## Layer 3 — The "Input values have N keys" error

A separate failure mode in the same cluster. Several nodes require items with exactly **one** JSON key:

- **Execute Sub-Workflow Trigger** in single-key input mode
- **AI Agent** and **Tools** nodes (LangChain wrappers expect a single `input` field)
- **Aggregate** node in *Aggregate Individual Fields* mode without a field specified

When you pass them multi-key items, they throw:

```
Input values have 3 keys, you must specify an input key or pass only 1 key as input
```

The fix is to reshape with a Set node before the receiving node — wrap the original multi-field object under one key (`{ "data": { id, name, email } }`). [`workflows/three-keys-error.json`](workflows/three-keys-error.json) reproduces this exactly.

## Why this is a top-3 forum complaint

Three reasons:

1. **The default Code node template uses `items[0]`-style examples** in some surfaces, which new users copy without realizing it's a single-item example.
2. **The error mode is silent for the loop case** — no exception, no warning, just missing data downstream. Users only notice when they count rows in their CRM and see 1 instead of N.
3. **Users blame Loop Over Items** because the symptoms feel "loop-shaped." Adding a Loop Over Items node doesn't help (the bug is in the Code node, not in iteration), which makes them blame n8n's iteration semantics. Both the [forum thread on first-item-only loops](https://community.n8n.io/t/loop-node-only-processes-the-first-item-how-to-fix/56727) and the [n8n staff "Have your say" thread on improving the Loop Over Items node](https://community.n8n.io/t/have-your-say-help-us-improve-the-loop-over-items-node/92570) trace back to this confusion.

## Forum citations

- [Loop Node only processes first item](https://community.n8n.io/t/loop-node-only-processes-the-first-item-how-to-fix/56727) — 6.8k views — symptom 1
- ["Input values have 3 keys"](https://community.n8n.io/t/input-values-have-3-keys-you-must-specify-an-input-key-or-pass-only-1-key-as-input/74622) — 7.3k views — symptom 4
- [Have your say: improve the Loop Over Items node](https://community.n8n.io/t/have-your-say-help-us-improve-the-loop-over-items-node/92570) — n8n staff thread acknowledging this is a recurring pain

## Upstream issue

A reasonable product-side improvement would be to update the default Code node starter snippet to `items.map(...)` in *Run Once for All Items* mode (or switch the default mode to *Run Once for Each Item*) and add an inline mode-switcher hint: *"This snippet processes every input item. To process the current item only, switch the Mode dropdown to Run Once for Each Item."* That's the kind of small docs/UX nudge that would deflect a measurable fraction of forum tickets in this cluster.
