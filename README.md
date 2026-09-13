# dynaprune

Observation-driven dynamic pruning for served language models. Status: design only.
Nothing here is implemented, deployed or measured beyond the cited prior art.

You supply three things. dynaprune turns them into a served artifact with a paper trail.

```
observations + model + budget/policy  →  pruning plan  →  derived, verified artifact + serving config
```

- **Observations** are inputs, not something dynaprune captures: routing mass, activation
  or saliency telemetry, per-unit error proxies, KL-contribution measurements, or any
  structured signal written against `schemas/observation.schema.json`.
- **Model** is a public, pinned precision tier of a checkpoint (for example an EXL3 branch).
- **Budget/policy** is a byte ceiling or prune fraction, a ranking function, structural
  clamps, and the list of control arms that must be built alongside.

"Dynamic" lives at the configuration layer. One core derives any point on the
(precision tier) × (non-uniform prune fraction) slider from the same inputs, and every
point is verified independently before it can be served. Quality is never assumed to be
monotonic in bytes; each point is measured and placed on a published frontier next to its
controls, including the points that lose.

## Roles

| Role | Owns | Reads | Writes |
|---|---|---|---|
| Observation provider | Capturing signals against a public model revision | the model | an observation set (`observation.schema.json`) |
| Planner | Ranking and byte-exact budget solving | observation set, policy | frozen plans, one per arm (`plan.schema.json`) |
| Deriver | Tensor surgery: re-pack, renumber, slice routers | source shards, plan | derived shards, manifest (`manifest.schema.json`) |
| Verifier | The measurement contract in `docs/METHOD.md` | derived artifact, sealed panels | receipts (`receipt.schema.json`) |
| Serving adapter | Translating plan + manifest into engine config | manifest | engine launch config (SGLang first, vLLM second) |
| Publisher | Immutable pins and the frontier | receipts, manifests | `frontier.json` (`frontier.schema.json`) |

The planner and deriver never see evaluation results. The verifier never chooses. The
publisher only records. That separation is the whole method.

## How the pieces relate

```
   observation      policy
   provider         (user)
       │               │
       ▼               ▼
  ┌──────────────────────────┐
  │ PLANNER  ingest→rank→solve│──▶ frozen plans: main + control arms
  └──────────────────────────┘        │  (hashed, read-only, pre-registered)
                                      ▼
  pinned source tier ─────────▶ ┌──────────┐
  (public revision + SHAs)      │ DERIVER  │──▶ derived shards + manifest
                                └──────────┘        │
                                                    ▼
             sealed eval panel ─────────▶ ┌────────────────────┐
             + teacher reference          │ VERIFIER  S3…S7     │──▶ receipts (append-only)
             + fixtures                   └────────────────────┘        │
                                                 │                      ▼
                                                 ▼               ┌───────────┐
                                        ┌────────────────┐       │ PUBLISHER │──▶ frontier.json
                                        │ SERVING ADAPTER│       └───────────┘
                                        │ sglang | vllm  │──▶ engine config for a point
                                        └────────────────┘
```

Arrows carry files that validate against a schema in `schemas/`. Every file is
content-addressed, so a point is reproducible by anyone holding the public source, the
plan, and the packer version. No account, key or contact with the author is required.

## Repository

| Path | What it is |
|---|---|
| `docs/METHOD.md` | The scientific method: hypotheses, control metrics, required controls, step sequence, fleet allocation, decision rules |
| `docs/ARCHITECTURE.md` | Language choice, file tree with line budgets, data contracts, pipeline stages, failure semantics, plugin boundary |
| `docs/OPEN-QUESTIONS.md` | Ranked unknowns |
| `schemas/` | JSON Schema (draft 2020-12) for observation, policy, plan, manifest, receipt, frontier |
| `BRIEF.md` | The design brief this repository answers |

## Prior art

This generalizes an instance design for one MoE model on one workstation. The instance
found that, at equal bytes on a weak base, uniform uncompensated expert pruning lost to
bit-demotion (61.8% vs 77.4% top-1). Non-uniform, saliency-ranked, compensated pruning on
a calibrated base is the open hypothesis dynaprune exists to test honestly. Academic
anchors: STEP-style bias compensation, DiEP/LAEP layer-adaptive rates, and Pollard-style
write-gain spread across layers as the argument for per-layer prune rates.

## License

MIT. Published artifacts carry no private hosts, paths, credentials, or unverified
performance claims.
