# METHOD — the scientific contract

Every slider point dynaprune serves is a measured claim. This document fixes what is
measured, against which controls, in what order, on which machine, and what the numbers
must show before a point's status changes. `docs/ARCHITECTURE.md` builds the machinery
that executes this contract; `schemas/receipt.schema.json` is where the numbers land.

Vocabulary: a **point** is (precision tier, frozen plan). A **tier** is a public, pinned
precision branch of a model. An **arm** is a point that exists for comparison. A
**stage** is one of S0–S7 below. **Status** climbs `planned → derived → candidate →
validated → recommended`, or drops to `rejected` / `quarantined`; it never skips.

## 1. Hypothesis framing

The question is not "does pruning work" but "at equal resident bytes, when does removing
units from a higher-precision tier beat demoting the whole model to a lower tier, and
for which users".

- **H0 (default, prior evidence)** At equal bytes, bit-demotion beats pruning. The
  instance work measured this on a weak base with uniform, uncompensated expert pruning:
  61.8% top-1 for the pruned arm against 77.4% for the demoted arm.
- **H1 (main)** Non-uniform, saliency-ranked, compensated pruning of a well-calibrated
  higher tier beats the equal-byte demoted tier on token fidelity (mean KL, top-1).
- **H2 (layer rates)** Per-layer prune rates driven by a layer signal (write-gain
  spread, reported as roughly 5–7.5× in early layers against 0.03–0.2× in late layers
  on a same-class model) beat uniform rates at the same total prune fraction.
- **H3 (compensation)** STEP-style bias compensation reduces the KL cost of a given
  removal set compared with removal alone.
- **H4 (audience)** Pruning and demotion damage different things. Demotion spreads
  small error over every token; pruning concentrates error on the tokens that routed to
  removed units. So token-fidelity users (KL, agreement, tail) and task-metric users
  (benchmarks, agentic acceptance) may prefer different points at the same bytes.

Each hypothesis is stated in the policy before any plan is frozen, and the frontier
records which points were built to test which hypothesis. A point that answers "no" is
published with the same care as one that answers "yes".

Two honesty constraints inherited from the instance work apply everywhere: with top-k
routing, pruning does not reduce per-token unit reads, so the only speed story is
resident bytes enabling a higher tier and it must be measured; and v1 prunes within a
tier because EXL3-family kernels take one scalar bit width per projection, so per-unit
bit mixing is out of scope.

## 2. Control metrics (the measurement contract)

All quality numbers are computed on a sealed evaluation panel (hashed inputs, fixed
tokenizer, fixed position count) that is disjoint from every observation panel. The
teacher is a sealed reference of full-vocabulary logits from the unquantized or highest
available tier, computed once and hashed. Candidate logits are produced teacher-forced on
the exact same token ids.

| Family | Metric | Definition | Where it lands |
|---|---|---|---|
| Quality | `kl_mean` | Mean over positions of KL(teacher ‖ candidate) over the full vocabulary, nats | `result.quality` |
| Quality | `kl_median`, `kl_p90` | Tail of the per-position KL distribution; catches concentrated damage that the mean hides | `result.quality` |
| Quality | `top1_agreement` | Fraction of positions where argmax matches the teacher | `result.quality` |
| Quality | `ppl` | Teacher-forced perplexity on the panel | `result.quality` |
| Quality | `kl_ci95` | Bootstrap CI over positions; two points whose CIs overlap are reported as tied | `result.quality` |
| Quality | `task_metrics` | Optional named task scores; recorded, never used for selection | `result.quality` |
| Degeneration | `loop_rate` | Fraction of fixed-payload generations that enter an exact token-id n-gram loop before the normal stop | `result.degeneration` |
| Degeneration | `acceptance_exact_ids` | Fraction of fixture payloads whose emitted ids equal the expected sequence | `result.degeneration` |
| Speed | prefill ladder | Prefill tokens/s at fixed input depths spanning the admitted context, cold prefix cache, ≥2 repeats per cell | `result.speed[]` |
| Speed | decode ladder | Total decode tokens/s at the same depths, windows >30 s, with and without speculative decoding when admitted | `result.speed[]` |
| Memory | `weights_resident_bytes` | Bytes of weights resident after load | `result.memory` |
| Memory | `kv_admission_tokens` | Largest single request admitted without OOM at the configured KV dtype | `result.memory` |
| Integrity | shard SHA receipts | Every source and output shard hashed; manifest totals match plan totals exactly | `result.integrity`, manifest |
| Integrity | N cold runs | Stages S3–S6 run N times from a cold process; N=3 by default; spread reported | `run_index` |
| Integrity | frozen plans | Plan hashed and read-only before S3; `selection_uses_heldout=false`; no evaluation number ever feeds back into a plan | plan |

Numbers that are not in this table are not selection inputs. Adding one is a change to
this document, not to a config file.

## 3. Required controls per comparison

A point is never compared to nothing. Every main arm is frozen together with:

| Arm | What it is | Why it exists |
|---|---|---|
| `equal_byte_demotion` | The lower tier, unpruned or pruned only as far as needed to reach the same byte count as the main arm | The H0/H1 test. Beating nothing else counts |
| `random_prune` | Same layer keep-counts as the main arm, ids drawn with a recorded seed | Separates "removing bytes" from "removing the right bytes" |
| `frequency_only` | Ranked by route count alone, same keep-counts | Tests whether anything beyond the cheapest signal matters |
| `unpruned_anchor` | The main arm's tier, unpruned, even if it does not fit the budget on the serving node | Ceiling for quality; measured on the quality node where bytes are not the constraint |

`uniform_prune` (same total fraction, equal per-layer rate) is added when H2 is under
test. All arms share the observation set, panel, teacher, fixtures, image digest and
engine version. The policy file lists the arms; the planner refuses to freeze a main plan
without them (`policy.controls`).

## 4. Step sequence

Each stage produces one receipt per run. Exit criteria are mechanical; a human never
"passes" a stage.
Receipts name the stage as `S<n>_<name>` (for example `S4_quality`), the enum in
`schemas/receipt.schema.json`.

| Stage | Name | Node | Entry | Exit (receipt `passed=true`) |
|---|---|---|---|---|
| S0 | Ingest | derive | observation set, policy, pinned source tier present | Observation validates against schema at a supported major; observation panel hash ≠ evaluation panel hash; every source shard SHA matches the pin; unit counts and bytes-per-unit parsed from safetensors headers, not estimated |
| S1 | Freeze | derive | S0 passed | Plans for main + every listed arm written; each keep-count is a multiple of `block` within `[layer_floor, layer_ceiling]`; `bytes.expected_total ≤ budget`; `selection_uses_heldout=false`; files hashed and made read-only. Status → `planned` |
| S2 | Derive | derive | S1 passed | Output shards hashed; per-layer unit counts equal the plan; `actual_total == expected_total`; router/mapping rows for retained ids equal source rows; protected tensors byte-identical; config patch present; index regenerated. Status → `derived` |
| S3 | Admit | serve | S2 passed, manifest imported and re-hashed on the serve node | Engine loads the artifact from the adapter's config; serves a request at the configured max context with retrieval fixtures correct; resident bytes and `kv_admission_tokens` recorded; no OOM across N cold runs. Status → `candidate` |
| S4 | Quality | quality | S2 passed, panel and teacher hashes match the lock | Full panel scored on exact position count; all metrics in §2 written; N cold runs agree within the bootstrap CI |
| S5 | Functional | serve | S3 passed | Fixture payloads run; `acceptance_exact_ids` and `loop_rate` written; multimodal fixtures run when the model has them |
| S6 | Speed | serve | S3 passed, node otherwise idle | Full prefill and decode ladders with repeats; decode windows >30 s; cells recorded with and without speculative decoding when admitted |
| S7 | Attest | attest | S2–S6 receipts present for the point and for every arm it lists | Attest node re-derives the artifact from the public source + plan + packer version and every output hash matches the manifest; a subset of S4 (≥10% of positions) reproduces within CI; a scrub finds no private hosts, paths or credentials in publishable files. Status → `validated`, then §6 decides `recommended` |

Order constraints: S4 may run in parallel with S3 because it needs the shards, not the
engine. S6 never overlaps another job on the serve node. S7 runs only on the attest node
and only after the point's arms have their own S2–S6 receipts.

## 5. Fleet allocation (4× DGX Spark, GB10, 128 GB unified, ARM64 sm121)

| Node | Role | Runs | Holds | Why this split |
|---|---|---|---|---|
| N1 | `derive` | S0, S1, S2; observation intake | Pinned source tiers, derived-artifact cache, plans | Derivation is CPU and disk bound and never needs the GPU. Keeping it off the measurement nodes means byte copying never perturbs a speed cell |
| N2 | `quality` | S4 | Sealed panel, teacher reference, candidate shards for the point under test | Teacher-forced scoring reads the whole artifact once per run and is throughput bound. The unpruned anchor can be scored here even when it exceeds the serving budget, because scoring can stream layers |
| N3 | `serve` | S3, S5, S6 | The pinned engine image, fixtures, one candidate at a time | Speed cells need an otherwise idle node with a stable thermal and memory state. Admission on this node is the definition of "fits" |
| N4 | `attest` | S7; nothing else | Public source tier pulled independently, plans, packer at pinned version, panel subset | Independence. N4 never receives derived shards; it rebuilds them from public inputs and compares hashes. A plan that only reproduces on the node that made it is not a plan |

The fourth node exists because reproducibility is a claim about other people's machines.
If N4 shares a cache, an image, or a filesystem with N1–N3, the S7 receipt proves
nothing. N4 is provisioned from the published lock file alone, the way a stranger would
be.

Throughput note: one point occupies N2 and N3 for roughly the wall-clock of N cold runs
of S4 and S3+S5+S6 respectively, so those two nodes are the pipeline's bottleneck. N1
can run ahead and queue derived points; N4 lags by design.

## 6. Decision rules

All comparisons use the point's own arms from §3, at the same image digest and engine
version, on the same panel hash. "Beats" means the 95% bootstrap CIs on `kl_mean` do
not overlap and the better point is also not worse on `top1_agreement` by more than the
CI half-width. "Ties" means the CIs overlap.

| From | To | Rule |
|---|---|---|
| `derived` | `candidate` | S3 passed on N3 with N cold runs |
| `candidate` | `validated` | S4, S5, S6 passed with N cold runs each; S7 passed on N4 (hashes match, subset reproduces, scrub clean); every listed arm is itself at least `candidate` with S4 complete |
| `validated` | `recommended` for `token_fidelity` | Beats `equal_byte_demotion` on `kl_mean`; `kl_p90` not worse than demotion by more than 25%; `loop_rate` not higher than the unpruned anchor's; `kv_admission_tokens` ≥ the policy's context; decode ladder not slower than demotion at any cell by more than the repeat spread |
| `validated` | `recommended` for `task_metrics` | Same as above with `task_metrics` (when present) replacing `kl_mean` as the primary, and `acceptance_exact_ids` not lower than demotion's |
| any | `rejected` | Ties or loses to `random_prune` on `kl_mean`, or loses to `equal_byte_demotion` on both primaries |
| any | `quarantined` | S7 hash mismatch, panel or observation hash drift, `lock_override=true`, or an S2 accounting mismatch. Quarantined points are listed on the frontier with the reason and cannot be served by default |

A `recommended` point is recommended relative to its arms, not in general. The frontier
shows the numbers; readers with a different metric can disagree with the label and the
data to do so is on the same line.
