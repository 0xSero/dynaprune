# BRIEF — dynaprune: observation-driven dynamic pruning (system design + scientific method)

You are designing, not deploying. Produce design documents and schemas.
"Minimal lines of code" is a hard requirement: prefer documents, JSON
Schemas, and thin interface definitions over implementation. Do not modify
anything outside `/Users/sero/projects/dynaprune`. No network. No
credentials. You may READ (read-only) the prior-art instance work at
`/Users/sero/dynamic-container/` (BRIEF.md, SCOPE.md, DESIGN.md when
present) — that is the GLM-specific instance of this general system.

## The idea (generalized)

A system where you take model(s) and configure what gets pruned based on
PROVIDED EXTERNAL OBSERVATIONS. Observations are inputs, not things this
system must capture: routing statistics, activation/saliency telemetry,
per-unit error proxies, KL-contribution measurements, or any structured
signal a user supplies against a published schema. The system's job:

  observations + model + budget/policy  →  pruning plan  →  derived,
  verified artifact + serving configuration

"Dynamic" at the configuration layer: the same core derives any point on
the (precision tier) × (non-uniform prune fraction) slider from the same
inputs, with every point independently verified before it can be served.

## Deliverables (write in this repo)

1. `README.md` — what this is, the roles (observation provider / planner /
   deriver / verifier / serving adapter / publisher), one diagram
   (ASCII), and how the pieces relate. Clean and simple.
2. `docs/METHOD.md` — the end-to-end SCIENTIFIC methodology:
   - Hypothesis framing: when does pruning beat bit-demotion at equal
     bytes, and for whom (token-fidelity vs task metrics)?
   - Control metrics (the measurement contract): quality — teacher-forced
     full-vocabulary KL, top-1 agreement, PPL, tail KL (median/p90),
     degeneration/loop metrics on exact token IDs; speed — decode and
     prefill ladders across context sizes; memory — resident bytes, KV
     admission; integrity — per-artifact SHA receipts, N cold runs,
     frozen plans with NO held-out feedback into selection.
   - Required controls per comparison: equal-byte demotion arm, random
     -prune control, frequency-only control, unpruned anchor.
   - Step sequence end to end, with entry/exit criteria per step.
   - A 4× DGX Spark (GB10, 128 GB unified, ARM64 sm121) fleet
     allocation: which node runs observations/derivation, quality
     evaluation, serving/speed, and independent reload + publication
     verification — and why the fourth node exists (independence).
   - Decision rules: what promotes a slider point from candidate to
     validated to recommended.
3. `docs/ARCHITECTURE.md` — the full system design:
   - Language selection WITH rationale (constraint: minimal code, ML
     ecosystem fit, schema-first; justify the choice and any secondary
     tooling).
   - Elements and files: a complete repo file tree where every file gets
     a one-line role. Include the intended line-count budget per file to
     enforce "minimal lines of code".
   - Data contracts: observation schema (source-agnostic, versioned),
     pruning-plan schema, budget/policy schema, derivation manifest +
     verification receipt schema. Draft these as JSON Schema files under
     `schemas/`.
   - Pipeline stages: ingest/validate → rank (pluggable saliency
     functions; built-ins: routed-mass, error-proxy, write-gain) →
     budget solver (byte-exact accounting, per-layer clamps, block
     granularity) → derivation (re-pack tensors, contiguous renumbering,
     router/mapping surgery — never logit masking) → verification
     harness (runs the METHOD contract) → serving adapter (SGLang first,
     vLLM second; adapters translate a plan into engine config) →
     publisher (immutable pins, anonymous-verifiable).
   - Failure semantics: partial derivations, verification failure,
     observation schema drift, plan non-reproducibility.
   - Extensibility: MoE experts now; attention heads/channels/layers
     later — show where the plugin boundary sits.
4. `docs/OPEN-QUESTIONS.md` — ranked unknowns.

## Prior art to anchor (from the instance work; cite, don't copy)

Equal-byte evidence on a weak base favored bits over uniform pruning
(61.8% vs 77.4% top-1); non-uniform, saliency-ranked, compensated pruning
on a calibrated base is the open hypothesis. Pollard-style write-gain
spread (early layers 5–7.5×, late layers 0.03–0.2×) justifies per-layer
rates. STEP-style bias compensation and DiEP/LAEP layer-adaptive rates
are the academic anchors. EXL3-family runtimes take one scalar K per
projection, so v1 prunes WITHIN a precision tier — per-unit bit mixing is
out of scope. Plans must be frozen before evaluation (pre-registration
discipline).

## Distribution constraints

MIT. No private IPs, SSH details, credentials, operator paths, or
unverified performance claims in any file destined for this public repo.
