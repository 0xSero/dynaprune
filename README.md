# dynaprune

Take a model. Take observations of which parts matter. Drop the parts that don't. Verify it still works.

```
observations + model + budget → pruned model
```

**This is a design.** The code doesn't exist yet — deliberately. One script, ~100 lines, when the first real use case demands it.

## How it works

1. **Observations in** — any JSON file mapping parts (experts, layers, heads) to importance scores. Routing mass, calibration error, KL contribution — whatever you measured.
2. **Budget picks** — given a byte budget, a greedy solver keeps the highest-scoring parts until the budget is full.
3. **Prune writes** — the output is a copy of the model with unimportant parts removed, router rows sliced, everything renumbered.
4. **Verify gates** — measure the pruned model against the original. If quality drops too much, the prune is rejected. No silent degradation.

## The one file that matters

```
dynaprune/
  README.md          ← this file
  prune.py           ← the script (when written)
```

## Rules

- Pruning is **irreversible** — always keep the original.
- Quality is **measured, not assumed** — every pruned model gets verified against the same test panel as the original.
- Selection uses **training data only** — test panels are for verification, never for choosing what to prune.
- The budget is **bytes, not percentages** — different parts have different sizes.

## Prior art this stands on

- Pollard (write-gain-weighted bit allocation across layers)
- brandonmusic (per-expert Hessian calibration for EXL3)
- turboderp (the quants we're pruning)

MIT license. See LICENSE.
