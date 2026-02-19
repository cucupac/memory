# Immutable Work Log

Raw, append-only design notes while iterating on the memory system.

Rules:
- Append only; do not rewrite prior entries.
- Corrections are new entries that reference the earlier timestamp.
- Keep entries terse and factual.

## 2026-02-18 01:40:38 CST

- Proposed: dogfood the memory design process by auto-recording notes without being asked.
- Worked: two-layer logging model (`immutable-work-log.md` raw + `discovery.md` abstraction) fits the current workflow.
- Did not work / risk: relying only on `discovery.md` loses raw context and intermediate reasoning.
- Open: how often to abstract raw notes into discovery (every turn vs milestone-only).
- Files touched: `insights/README.md`, `insights/00-lab/immutable-work-log.md`, `insights/discovery.md`.

## 2026-02-18 01:45:54 CST

- Proposed: decide between formal Bayesian updates and lightweight heuristic updates for truth/utility.
- Worked: framing as a staged approach preserves velocity while keeping Bayesian compatibility.
- Did not work / risk: full Bayesian formalization now is likely overfit/over-complex given low-volume evidence.
- Decision (provisional): use lightweight heuristic updates in v1; design signals so Bayesian updating can be added later without schema or ontology churn.
- Cadence decision: append to raw log every ideation turn; add to `discovery.md` only when there is a decision, policy shift, or new open-question cluster.
- Open: explicit thresholds for when to graduate from heuristics to Bayesian (for example, minimum observations per memory/problem neighborhood).
- Files touched: `insights/00-lab/immutable-work-log.md`, `insights/discovery.md`.

## 2026-02-18 01:55:00 CST

- Proposed: use LLM certainty plus supporting evidence references for truth/utility updates.
- Worked: lightweight rule can require evidence without introducing rigid taxonomy.
- Did not work / risk: requiring strict evidence for every utility signal may be too heavy in low-friction workflows.
- Open: should truth updates require hard evidence while utility updates allow softer evidence from session outcomes?
- Files touched: `insights/00-lab/immutable-work-log.md`.

## 2026-02-18 01:59:00 CST

- Proposed: utility evidence should be optional; truth evidence should be strict.
- Worked: reducing strict attribution burden for utility preserves practicality.
- Did not work / risk: utility updates without evidence can drift or overfit if unconstrained.
- Proposed interface direction: use a clean scoped memory API with JSON payloads/returns rather than freeform text parsing.
- Open: best interaction form for the model layer (tool-style API, CLI wrapper, or both).
- Files touched: `insights/00-lab/immutable-work-log.md`.

## 2026-02-18 02:24:04 CST

- Proposed: consolidate status and define concrete next implementation decisions.
- Worked: explicit request to move from ideation summary to concrete spec targets.
- Did not work / risk: delaying concrete contract decisions may stall transition to implementation.
- Open: choose exact API verbs/payloads, update function shape, and evidence reference contract.
- Files touched: .


## 2026-02-18 02:24:40 CST (Correction)

- Correction for entry `2026-02-18 02:24:04 CST`: malformed `Files touched` field due shell interpolation in append command.
- Correct files touched: `insights/00-lab/immutable-work-log.md`.

## 2026-02-18 02:30:00 CST

- Proposed: make categorization explicit via write-contract choices (scope + kind) with LLM judgment plus lightweight validation.
- Worked: concrete JSON shape for `read` and `write` reduces ambiguity about how categorization happens.
- Did not work / risk: too many enum choices could reintroduce rigidity if not kept minimal.
- Open: final canonical kind set (fact/preference/tactic vs experiential subtypes) and auto-vs-explicit override behavior.
- Files touched: `insights/00-lab/immutable-work-log.md`.

## 2026-02-18 02:32:59 CST

- Proposed: clarify update API semantics and remove ambiguous auto categorization in write payloads.
- Worked: distinguish optional two-step update (propose/apply) from simpler one-step commit flow.
- Did not work / risk: auto categorization field name created confusion about where judgment lives.
- Open: whether v1 uses two-step approval flow or single update endpoint with dry_run/commit mode.
- Files touched: insights/00-lab/immutable-work-log.md.

## 2026-02-18 02:38:22 CST

- Proposed: settle v1 interface semantics: write creates immutable memories (including change/update memories), update adjusts dynamic values (truth/utility).
- Worked: clear separation resolves confusion between "update memory" and "update operation".
- Did not work / risk: if update operation is treated as hard overwrite, provenance can be lost; recommend event-backed projection under the hood.
- Open: exact allowed kind enum and required evidence fields by operation.
- Files touched: insights/00-lab/immutable-work-log.md.

## 2026-02-18 02:47:58 CST

- Proposed: persist approved interface schemas as a concrete contract card and link from discovery.
- Worked: captured exact approved JSON schemas for read/write/update in a single source-of-truth card.
- Did not work / risk: none noted in this step.
- Open: semantic validation matrix details by kind remain to be finalized.
- Files touched: insights/04-contracts/memory-interface-json-schemas-v1.md, insights/discovery.md, insights/README.md, insights/00-lab/immutable-work-log.md.
