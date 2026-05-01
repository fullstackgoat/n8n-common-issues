# Repro 07 — Loops, iteration & data-shape errors

> ⏳ **Planned.** See [`repros/01-webhook-url/`](../01-webhook-url/) for the pattern this folder will follow.

## What this will reproduce

The most teachable category — almost always user-error and almost always KB-able.

Workflows shipped in this repro will demonstrate:

- "Loop Over Items" only processing the first item
- "Input values have 3 keys, you must specify an input key"
- Undefined data after a loop
- API that 400s only on the second iteration of a loop because the prior payload mutated state
- Set node leaking state between iterations

## Source threads

- [Loop Node only processes first item](https://community.n8n.io/t/loop-node-only-processes-the-first-item-how-to-fix/56727) — 6.8k views
- ["Input values have 3 keys"](https://community.n8n.io/t/input-values-have-3-keys-you-must-specify-an-input-key-or-pass-only-1-key-as-input/74622) — 7.3k views
- [How to check for empty node output](https://community.n8n.io/t/how-to-check-for-empty-node-output/10022) — 24.4k views
- [API 400 only on 2nd loop iteration](https://community.n8n.io/t/help-evolution-api-throws-400-bad-request-object-object-only-on-the-2nd-item-inside-an-n8n-loop/293177)
- [Loop can't handle multiple incoming lists](https://community.n8n.io/t/loop-over-items-cant-handle-multiple-incoming-lists/234133)
- ["Help us improve Loop Over Items" — n8n staff thread](https://community.n8n.io/t/have-your-say-help-us-improve-the-loop-over-items-node/92570)
