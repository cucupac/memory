# Discovery Log

Chronological, append-only record of how memory-system ideas evolved.

## 2026-02-14 - Foundation Baseline

Conversation established the baseline architecture captured in:
- `01-foundation/summary.md`
- `01-foundation/architecture-insights.md`
- `01-foundation/memory-ontology.md`
- `01-foundation/memory-creation-and-categorization.md`
- `01-foundation/storage-layer.md`

Key outcomes:
- Interface boundary set to `read`, `write`, `dispute`.
- Scope simplified to repo + global only (no domain layer).
- Retrieval pipeline set to scope -> relevance -> association -> threshold -> selection.
- Storage anchored on SQLite with immutable event log authority and rebuildable projections.
- Principle set to precision over recall, including "empty is correct" when confidence is low.

## 2026-02-18 - Outside-In Breakthrough Pass

Raw ideation captured in:
- `02-breakthrough/breakthrough-raw.md`

Structured reformulation captured in:
- `02-breakthrough/breakthrough-structured.md`

Key developments:
- Retrieval clarified as two-lane recall: semantic association lane + bounded keyword lane, then dedupe/merge.
- Strong emphasis that codebase state is immediate ground truth and memory is advisory utility.
- Session-end write flow clarified with duplicate checks, contradiction handling, and optional autonomous storage mode.
- Problem-solution linkage became explicit: retrieval and expansion operate over linked memories.

## 2026-02-18 - Failed Tactics Promoted To First-Class Memory

Design extension incorporated into structured and core docs:
- `02-breakthrough/breakthrough-raw.md`
- `02-breakthrough/breakthrough-structured.md`
- `01-foundation/architecture-insights.md`
- `01-foundation/memory-ontology.md`
- `01-foundation/memory-creation-and-categorization.md`
- `01-foundation/storage-layer.md`
- `01-foundation/summary.md`

Key developments:
- Failed attempts are no longer implicit only in episode text.
- Experiential memory is now modeled as linked `problems`, `solutions`, and `failed_tactics`.
- Immutable episode logs remain evidence, while normalized linked records power retrieval.

## 2026-02-18 - Utility vs Truth Framing (User Verbatim Points)

Verbatim source notes recorded in:
- `03-refinements/user-three-points-verbatim.md`

Synthesis of the three-point direction:
- Distinguish instantaneous truth (current code state) from historical memory utility.
- Track memory usefulness relative to specific problem contexts, not only global vote totals.
- Treat changes in utility and changes in truth as separable update streams over immutable history.

## 2026-02-18 - Policy Concretization: Overfitting, Weak Priors, and Chain Classes

Detail note added:
- `03-refinements/policy-overfitting-bayesian-chains.md`

What changed:
- Adopted a concrete policy against stale-memory overfitting: memory is optional utility, and current workspace/code observation is the instantaneous truth source.
- Confirmed global utility can exist as a weak prior and is best treated as computed-after-the-fact rather than primary.
- Recognized a shared Bayesian flavor between utility and truth updates:
  - utility as a belief updated by context-specific outcomes,
  - truth as a belief in `[0,1]` updated by supporting/contradicting evidence.
- Clarified "chain idea" now has two distinct forms:
  - implicit association chain via semantic/vector similarity,
  - explicit association chain via formal deterministic links (for example, base memory -> update memories).

Why it changed:
- To keep the system aligned with optionality and avoid rigid deterministic handling that drifts from the original design intent.
- To preserve immutable history while still enabling "what is useful now" and "what is likely true now" behavior.

What remains open:
- Whether to formally implement Bayesian update math or keep the same concept with lightweight heuristics.
- Exact weighting mechanics for weak global priors vs problem-specific evidence.
- Prompt wording and retrieval behavior that enforce explicit-association traversal without over-constraining exploration.

## 2026-02-18 - Dogfooding Workflow: Raw Log + Discovery Abstraction

What changed:
- Added a dedicated immutable raw-note log: `00-lab/immutable-work-log.md`.
- Clarified that `discovery.md` is an abstraction/extraction layer over raw notes.
- Updated structure guidance so agents should record notes automatically, without waiting for explicit prompts.

Why it changed:
- To dogfood the memory design process while designing it.
- To preserve intermediate reasoning and dead ends, not only polished summaries.

What remains open:
- Cadence for abstraction from raw log into discovery (every significant turn vs milestone-only).

## 2026-02-18 - Provisional Decision: Heuristics First, Bayesian-Compatible Later

What changed:
- Set abstraction cadence: raw log updates every ideation turn; discovery updates when there is a decision, policy shift, or a coherent new open-question cluster.
- Took a provisional stance on update mechanics: lightweight heuristics first for truth/utility updates, with Bayesian compatibility preserved for later.

Why it changed:
- Current evidence volume is low, so full Bayesian machinery now risks complexity before signal quality justifies it.
- A heuristic-first path keeps momentum while still preserving the conceptual Bayesian framing already recognized in this project.

What remains open:
- Graduation criteria from heuristic to Bayesian updates (for example, observation count, calibration error, or drift frequency thresholds).
- Exact heuristic forms for truth update and utility update in v1.

## 2026-02-18 - Ratified Direction: JSON Contracts and Evidence Strictness Split

What changed:
- Ratified interface direction: use a clean scoped memory API/tool surface with JSON request/response contracts.
- Ratified evidence policy split:
  - truth updates require supporting evidence references,
  - utility updates allow optional evidence.
- Ratified update style: keep lightweight heuristic updates in `[0,1]` now, while preserving a path to Bayesian formalization later.

Why it changed:
- JSON contracts improve reliability for LLM interaction without over-constraining reasoning behavior.
- Strict truth evidence protects against drift; optional utility evidence avoids heavy attribution overhead in low-friction workflows.
- Heuristic-first keeps momentum during ideation and low-volume operation.

What remains open:
- Exact scoped API verbs and payload fields.
- Fallback behavior when utility updates have no evidence (for example, smaller deltas or deferred persistence).
- Concrete v1 heuristic update function details.

## 2026-02-18 - Ratified Interface Semantics: `read`, `write`, `update` + Validation Layer

What changed:
- Ratified interface operation split:
  - `read`: retrieval only.
  - `write`: creates immutable memory records.
  - `update`: adjusts dynamic values on existing memory IDs (`truth`, `utility`) in `[0,1]`.
- Ratified interpretation for "update memories":
  - Statements like "X changed in the codebase and invalidates Y" are new memories and therefore use `write` (kind `change`), not `update`.
- Ratified memory kind set for v1 interface contracts:
  - `problem`, `solution`, `failed_tactic`, `fact`, `preference`, `change`.
- Ratified categorization rule:
  - The writing agent provides explicit `scope` and `kind` (no ambiguous `auto` categorization in payloads).
- Ratified validation placement:
  - A validation layer sits directly under the interface.
  - It performs schema validation (JSON shape/types/required fields) and semantic validation (domain rules such as evidence requirements).

Why it changed:
- This split resolves ambiguity between immutable memory creation and mutable value adjustment.
- It preserves append-only memory history while still allowing current utility/truth projections to evolve.
- It makes agent behavior parseable and auditable without over-constraining reasoning.

What remains open:
- Exact canonical JSON payload definitions and required-field matrix by operation.
- Final semantic validation rules by memory kind (for example, required links for `solution` and `failed_tactic`).

## 2026-02-18 - Ratified Card: Exact JSON Schemas for `read` / `write` / `update`

Card added:
- `04-contracts/memory-interface-json-schemas-v1.md`

What changed:
- Recorded the exact approved JSON schemas verbatim for:
  - `memory.read.request`
  - `memory.write.request`
  - `memory.update.request`
- Confirmed operation semantics in the card:
  - `write` creates immutable memories,
  - `update` adjusts dynamic values,
  - \"update memory\" statements are `write` with `kind: change`.
- Confirmed validation placement directly under the interface (schema + semantic validation).

Why it changed:
- To make the interface concrete and implementation-ready without ambiguity.
- To preserve the approved contract in a single source-of-truth card that discovery can link to.

What remains open:
- Final semantic validation matrix by kind/field requirement (especially linking requirements and edge-case rules).

## Update Rule

When a new memory-design idea is discussed:
- First append raw notes to `00-lab/immutable-work-log.md`.
- Then append a distilled dated entry to `discovery.md` with:
- What changed.
- Why it changed.
- Which files were updated.
- What remains open.
