# ARCHITECTURE

The system that executes `docs/METHOD.md`. Design goal in one line: **schemas carry the
meaning, code only moves bytes.** Every stage boundary is a file that validates against
`schemas/`, so each stage can be re-run, audited or replaced without reading the others.

## 1. Language selection

**Python 3.11+, with JSON Schema (draft 2020-12) as the primary design artifact.**

| Constraint | Why Python satisfies it | What was rejected and why |
|---|---|---|
| Minimal code | `safetensors` header parsing, tensor slicing, `jsonschema` validation and hashing are each a few lines from stdlib or one dependency | Rust: fastest safe byte copying, but every schema becomes a struct and every stage a crate; 3–5× the lines. Go: no tensor ecosystem |
| ML ecosystem fit | Engines (SGLang, vLLM), tokenizers, `safetensors`, `numpy`/`torch` and evaluation harnesses are Python-native; adapters call their launch code directly | TypeScript: fine for schemas, absent from the serving side |
| Schema-first | `jsonschema` with the `referencing` registry resolves the cross-file `$ref`s in `schemas/` unchanged; the same files document, validate and generate | Pydantic-as-source-of-truth: schemas would be derived from code, inverting the priority |

Secondary tooling, each justified by removing code rather than adding it:

- **`safetensors`** (header read, zero-copy slice): the deriver is a header rewrite plus
  contiguous byte copies, so no framework tensor object is needed for surgery.
- **`numpy`** for ranking and the solver; no torch dependency in planner or deriver.
- **`jq` + `sha256sum`** in shell for receipts and scrubs on the attest node, so N4 can
  verify with nothing installed but coreutils and a JSON tool.
- **Container image, digest-pinned**, carrying the engine. dynaprune never patches an
  engine; a point that needs an engine feature the pinned image lacks is unservable there.

Deliberately absent: a web service, a database, a plugin marketplace, a config DSL.
Files in directories, hashed, are the database.

## 2. Elements and files

Line budgets are hard ceilings for the future implementation and are counted excluding
blank lines and comments. A file over budget is a design bug, not a code review note.

```
dynaprune/
├── README.md                     roles, diagram, how the pieces relate           (≤ 100)
├── LICENSE                       MIT                                              (   21)
├── BRIEF.md                      the design brief (input, untouched)
├── docs/
│   ├── METHOD.md                 measurement contract, stages, fleet, decisions   (≤ 160)
│   ├── ARCHITECTURE.md           this file                                        (≤ 260)
│   └── OPEN-QUESTIONS.md         ranked unknowns                                  (≤  80)
├── schemas/                      the data contracts; the real source of truth
│   ├── observation.schema.json   source-agnostic per-unit signals, versioned      (≤ 120)
│   ├── policy.schema.json        budget / ranking / clamps / controls             (≤  90)
│   ├── plan.schema.json          frozen retained-id sets + byte accounting        (≤  70)
│   ├── manifest.schema.json      derived shards ↔ source + plan + packer          (≤  90)
│   ├── receipt.schema.json       one stage run's numbers and hashes               (≤ 120)
│   └── frontier.schema.json      published list of points with status            (≤  40)
├── pyproject.toml                package metadata, 4 runtime deps                  (≤  30)
├── lock.json                     pinned image digest, engine version, panel/teacher/fixture hashes  (data)
├── dynaprune/                    the implementation (total ≤ 1,000 lines)
│   ├── __init__.py               version string                                   (≤   5)
│   ├── cli.py                    one subcommand per stage + serve/publish; arg → stage function  (≤  90)
│   ├── contracts.py              load schema registry; validate(obj, name); canonical_json; sha256  (≤  50)
│   ├── ingest.py                 S0: validate observation + policy, pin check, header byte census  (≤  70)
│   ├── rank.py                   ranker registry + built-ins routed_mass, error_proxy, write_gain, random, frequency  (≤  90)
│   ├── solve.py                  S1: layer-rate policy → per-layer keep counts → block-quantized byte-exact plan; emits arms  (≤ 110)
│   ├── derive.py                 S2: stream shards, call unit family repack, write manifest; resumable  (≤ 120)
│   ├── verify.py                 S3–S7 drivers: run a harness step, collect numbers into a receipt  (≤ 110)
│   ├── decide.py                 METHOD §6 rules as pure functions over receipts → status  (≤  70)
│   ├── publish.py                append to frontier.json, immutable pin layout, scrub  (≤  60)
│   ├── units/                    plugin boundary: what a "unit" is for one model family
│   │   ├── base.py               UnitFamily protocol (enumerate, bytes_per_unit, repack, config_patch)  (≤  40)
│   │   └── moe_expert.py         v1: expert tensors + router rows + correction bias; contiguous renumbering  (≤ 120)
│   └── adapters/                 plugin boundary: plan + manifest → engine launch config
│       ├── base.py               ServingAdapter protocol (requirements, launch_args, probe)  (≤  30)
│       ├── sglang.py             first target                                     (≤  50)
│       └── vllm.py               second target                                    (≤  50)
├── harness/                      thin wrappers over existing eval tools; no metric logic re-implemented
│   ├── quality.py                S4: teacher-forced full-vocab KL / top-1 / PPL / tail on the sealed panel  (≤  90)
│   ├── functional.py             S5: fixture payloads, exact-id acceptance, loop detection  (≤  60)
│   ├── speed.py                  S6: prefill/decode ladders                       (≤  60)
│   └── attest.sh                 S7: re-derive, re-hash, subset-score, scrub; coreutils + jq only  (≤  60)
└── tests/
    ├── test_schemas.py           every schema compiles; fixtures validate; negative cases  (≤  60)
    ├── test_solve.py             byte-exactness, block alignment, clamps, arm generation  (≤  80)
    └── test_derive.py            synthetic 2-layer model: renumbering, router slicing, protected-tensor identity  (≤  80)
```

Implementation total: ≤ 1,000 lines in `dynaprune/`, ≤ 270 in `harness/`, ≤ 220 in
`tests/`. Documents and schemas are expected to outweigh code by roughly 2:1.

## 3. Data contracts

All six are in `schemas/`. Each carries `schema_version` (major.minor); consumers
refuse an unknown major and warn on a newer minor. Every top-level object is
content-addressed: its id is the SHA-256 of its canonical JSON with the id field blank.

| Contract | Produced by | Consumed by | Key invariants |
|---|---|---|---|
| `observation` | provider | ingest, rank | Units addressed by `(unit_type, layer, index)` in source numbering; every signal is a dense `[layer][unit]` matrix; `panel.sha256` must differ from the evaluation panel; `custom` kinds carry a definition |
| `policy` | user | solve | Exactly one of `budget_bytes` / `prune_fraction`; `layer_floor` a multiple of `block`; `controls` lists the arms; `ranking.function` names a registered ranker |
| `plan` | solve | derive, verify, publish | `retained_ids` ascending and unique per layer; `keep_count` a multiple of `block`; `bytes.expected_total = source_total − removed × bytes_per_unit`; `selection_uses_heldout` is the constant `false`; `role` says which arm |
| `manifest` | derive | verify, adapters, publish | Output shard hashes; `actual_total == expected_total`; `protected_tensors_identical`; `mapping_rows_verified`; `config_patch` has per-layer counts and ids, never a mask; `status` partial/complete/failed |
| `receipt` | verify (each stage) | decide, publish, attest | Stage and `node_role` enumerated; failed receipts require a reason; numbers only, no hosts or paths; `lock_override=true` blocks publication |
| `frontier` | publish | readers, adapters (default slider) | Append-only; every point lists its arms and receipt ids; status is the only judgement; no "best" flag |

## 4. Pipeline stages

```
S0 ingest/validate → S1 rank + budget solve → S2 derive → S3–S6 verify → S7 attest → publish
      (N1)                    (N1)               (N1)        (N2, N3)        (N4)      (N1)
```

**S0 Ingest/validate** (`ingest.py`). Validate observation and policy against the
registry. Check the source tier's shard SHAs against `lock.json`. Parse safetensors
headers to build a byte census: per-unit packed bytes, per-unit mapping bytes (router
rows, correction-bias entries), non-prunable bytes. The census, not an estimate, is what
the solver uses. Refuse if the observation panel hash equals any evaluation panel hash.

**S1 Rank** (`rank.py`). A ranker is a pure function
`score(observation, unit_universe, params) → [layer][unit] keep-priority`. Built-ins:

| Ranker | Score | Signals required | Reading |
|---|---|---|---|
| `routed_mass` | Σ router probability over the panel | `routed_mass` | Keep what the router uses most |
| `error_proxy` | Estimated loss increase if the unit is removed with compensation: `route_count × (output error of the unit's best replacement)` | `route_count`, `error_proxy` or `kl_contribution`, optional `mean_output` | Keep units whose absence hurts, not merely units that are busy |
| `write_gain` | `routed_mass × write_gain` per unit; also supplies the per-layer aggregate used by `layer_rate.mode = write_gain_proportional` | `routed_mass`, `write_gain` (unit or layer level) | Keep units that write strongly into the residual; prune late layers harder where write gain collapses |
| `frequency` (control) | `route_count` | `route_count` | Cheapest possible signal |
| `random` (control) | seeded uniform | none | Bytes-only baseline |

Rankers never see bytes or budgets; they only order.

**S1 Budget solver** (`solve.py`). Inputs: ranked scores, byte census, policy. Steps:
1. Resolve the target. A `budget_bytes` target becomes the smallest block-quantized
   removal count `r` with `source_total − r × bytes_per_unit ≤ budget − reserve`.
2. Set per-layer keep counts by `layer_rate.mode`: `uniform` (equal fraction),
   `global_rank` (greedy removal of the lowest-scored unit anywhere until `r`, blocks of
   `block`), `write_gain_proportional` (rates ∝ `layer_signal^-exponent`, normalized to
   `r`), or `explicit`. Clamp every layer to `[layer_floor, layer_ceiling]`, respect
   `protected_layers` and `protected_units`, and re-distribute any shortfall by
   continuing the greedy pass over unclamped layers.
3. Within each layer, retain the top `keep_count` by score; sort retained ids ascending.
4. Recompute `expected_total` from the census. Assert equality with the target within
   one block; otherwise fail S1 with the shortfall in the receipt.
5. Emit the main plan and one plan per arm in `policy.controls` (same keep counts for
   `random_prune` and `frequency_only`; a re-solved budget on the lower tier for
   `equal_byte_demotion`; keep-all for `unpruned_anchor`). Hash, write read-only.

**S2 Derivation** (`derive.py` + `units/moe_expert.py`). For each source shard, stream
tensors. Protected tensors are copied byte-identical. For prunable tensors the unit family
plugin re-packs: retained unit slices concatenated in ascending source-id order, so new
id `j` is `retained_ids[j]` (contiguous renumbering). Router and mapping surgery slices
the router weight rows and correction-bias entries to the retained ids in the same order;
router logits are never masked. Compensation, when the plan names it, adds the
mass-weighted mean output of removed units to the layer's shared-path bias (STEP-style)
and optionally renormalizes retained router rows. The config patch records
`routed_units_per_layer` and `retained_unit_ids_by_layer`. Output shards are re-sharded
to the source shard size, hashed, and the manifest written. Writes go to a staging
directory and are renamed into place atomically; the manifest's `status: partial` lists
completed shards so a restart resumes.

**S3–S6 Verification harness** (`verify.py` + `harness/`). Each driver runs one METHOD
stage for one point and writes exactly one receipt per cold run. The harness re-uses
existing measurement tooling behind a thin wrapper; dynaprune owns only the contract
(which numbers, on which hashes) and the receipt.

**Serving adapter** (`adapters/`). `ServingAdapter.requirements()` lists what the engine
must support (`per_layer_unit_counts`, quantization format, KV dtype).
`launch_args(manifest, policy.serving, lock)` returns the engine's argument list; the
receipt stores its hash. `probe(url)` runs the S3 admission request. SGLang is the first
adapter because the instance overlay already loads per-layer expert counts from config;
vLLM is second and its adapter fails fast until that loader exists. Adapters never
modify weights.

**S7 Attest + Publisher** (`harness/attest.sh`, `publish.py`). N4 pulls the public
source, re-runs `derive` from the plan at the pinned packer version, and compares every
output hash to the manifest. It scores a panel subset and diffs against the S4 receipt.
It scrubs every publishable file for hostnames, IPs, absolute paths and secrets.
`publish.py` then lays out the immutable pin:

```
frontier/<model_tag>/<point_id>/{plan.json, manifest.json, receipts/*.json}
frontier/<model_tag>/frontier.json
```

Anyone can verify a pin anonymously: fetch the public source at the pinned revision,
run `derive` with the plan, and compare hashes. No weights need be distributed; mirrored
weights are a cache, never the source of truth.

## 5. Failure semantics

| Failure | Detection | Response |
|---|---|---|
| Partial derivation (crash, disk full, interrupt) | Manifest `status: partial`; staging directory present | Re-run S2: completed shards are re-hashed and skipped if they match; the rest are rebuilt. Nothing partial is ever moved into the artifact cache |
| Verification failure (any of S3–S6) | Receipt `passed=false` with reason | Point keeps its current status. Receipt is appended, never deleted. Re-running is allowed only with a new `run_index`; the failed run stays on record. Three consecutive failures at one stage mark the point `rejected` |
| Observation schema drift | `schema_version` major mismatch, or a signal shape not matching `units` | S0 refuses. Minor drift warns and records the version in the plan's inputs. Migrations are explicit files under `schemas/migrations/` when they exist; there is no silent coercion |
| Plan non-reproducibility | S7 output hashes differ from the manifest, or S1 re-run on N4 yields a different `plan_id` from the same inputs | Point → `quarantined` with the differing hashes recorded. Cause is investigated as a packer or ranker determinism bug; the point can be rebuilt only with a new packer version, producing a new point id |
| Engine cannot honor the config patch | `ServingAdapter.requirements()` unmet, or S3 load error | Point is unservable on that engine; S3 receipt says so. Masking is never used as a fallback |
| Panel or teacher hash drift | Receipt `inputs.panel_sha256` differs from `lock.json` | Receipt is written with `lock_override=true` and cannot feed a status change or publication |
| Budget cannot be met within clamps | S1 solver shortfall | S1 fails with the minimum achievable bytes in the receipt; the user changes the policy. The tool never auto-relaxes a floor |

## 6. Extensibility: where the plugin boundary sits

Two protocols, both in `base.py` files, are the only places new model structure enters.

**`UnitFamily`** (`units/base.py`) answers four questions for one kind of prunable unit:

| Method | v1 `moe_expert` | Later `attention_head` | Later `mlp_channel` | Later `layer` |
|---|---|---|---|---|
| `enumerate(config, headers)` → layers × unit count | routed experts per MoE layer | heads per attention layer | intermediate channels | layer indices |
| `bytes_per_unit(headers)` | expert tensor bytes + router row + bias entry | q/k/v/o slices per head (GQA-aware) | one column of up/gate + one row of down | whole layer |
| `repack(tensors, retained_ids)` | concatenate expert slices; slice router rows | slice head dims; adjust `num_heads` | slice channels; adjust `intermediate_size` | drop tensors; renumber layers |
| `config_patch(plan)` | per-layer counts + ids | per-layer head counts | per-layer intermediate sizes | new layer count + map |

Everything above the protocol (ingest, rank, solve, verify, decide, publish) is unit
agnostic: the observation schema already takes any `unit_type`, and the solver works on
`[layer][unit]` scores and a bytes-per-unit census. A new family is one file plus one
enum value in `observation.schema.json`. Mixed families in one plan are out of scope
until the solver can take a bytes vector instead of a scalar.

**`ServingAdapter`** (`adapters/base.py`) isolates engine knowledge. A new engine is one
file that knows which config keys and launch flags its engine reads. Adapters cannot see
observations, scores or receipts, so no engine can influence selection.

Rankers are the third, softer boundary: a name in `policy.ranking.function` resolved by a
registry in `rank.py`. Custom rankers ship as an importable module and are recorded by
name and version in the plan's inputs.
