## Proposal v4 (staged nucleus)

**Thesis**: ship the compounding core first: **immutable log + evidence-addressed cards + scoped retrieval + consolidation budgets + compaction**. Treat causal attribution as an instrumentation upgrade, not v1 functionality.

---

## 0) What v4 optimizes for

**Goal**: in a low-volume single-user regime, deliver compounding by (a) preventing repeated mistakes, (b) persisting constraints and preferences with citations, and (c) reusing tactics that empirically worked, without needing fine-grained causal inference.

**Main failure mode to avoid**: card-store entropy (too many low-signal cards) degrading retrieval.

---

## 1) Data model (minimal, rebuildable)

### EpisodeLog (append-only, canonical)

**Episode**: one interaction run. Stores raw user text, model text, tool outputs, stable IDs, hashes.

**EvidenceRef**: pointer into EpisodeLog or external artifact:
`{kind: user_span|tool_output|doc_span, id, start, end, hash}`

### CardStore (lossy index, rebuildable)

**Card**:
`{id, kind, statement, scope_tier, scope_id, tags, embedding, status, supersedes_id?, topic_key, created_from:[EvidenceRef...], created_at, rule_version}`

**Kinds** (small set, different epistemology):

* **Preference / Constraint / Commitment** (normative): active only if anchored to user_span; changes only by supersession.
* **Fact** (descriptive): active only if anchored to user_span/doc_span/tool_output; truth monotone in anchored evidence.
* **Tactic** (policy): value measured empirically; not treated as fact.
* **NegativeResult** (guardrail): anchored to failure evidence; retrieval priority inside check().

### Exposure (simple, required)

**Exposure**: `(episode_id, card_id, channel, shown_at)`
Channel is `auto_pack|search|explicit_read|check`.

This is the only independence tracking in v4; no DecisionPoints.

---

## 2) Consolidation (rate-controlled and observable)

Consolidation is a post-episode process that takes raw `log(event, evidence_refs)` events plus episode text and proposes candidate cards.

### Candidate generation (can be fuzzy)

Generate candidates for:

* explicit user preferences/constraints/commitments
* explicit facts derived from anchored spans
* tactics when the agent produces a reusable procedure
* negative results from tool outputs indicating failure

### Acceptance is deterministic under budgets (no “expected reuse”)

Admit candidates only if:

* they have required EvidenceRefs (per kind), and
* they pass a novelty threshold vs existing cards in the same (kind, scope), and
* the per-(scope_tier, kind) **budget** is respected.

**Budgets**

* Hard cap per (scope_tier, kind) and soft cap per episode delta.
* Example principle: repo scope can be denser than domain/global.

### Merging and supersession

* Near-duplicates merge deterministically (embedding + lexical overlap).
* Preferences/constraints supersede only via later user_span evidence.
* Facts merge evidence sets; contradictions handled separately.

### Consolidation ledger (mandatory)

For every episode:
`N proposed, M admitted, K merged, J superseded, A archived` + reasons by rule.
This is the main debugging surface.

---

## 3) Truth, dispute, and reverify (no decay, but no hiding)

### Facts

**TruthStats** are derived from evidence:

* evidence_count_by_kind
* anchored_contradiction_count_by_kind

**Rule**: only anchored contradictions affect Truth. Unanchored conflicts become disputes.

### DisputeEdge (simple)

**DisputeEdge**: `(card_id, evidence_ref, created_at, weight)`
Weight depends on evidence kind (tool_output > doc_span > user_span).

### needs_recheck scheduling (minimal, no volatility field)

Instead of a free volatility number, use a small fixed mapping:

* repo-local facts: fast recheck
* domain facts: medium
* global facts: slow

When dispute mass crosses a threshold for that tier, set `status = needs_recheck`.

### Retrieval effect

needs_recheck cards are downranked in auto_pack, but returned in explicit search/read with a “needs recheck” flag and citations.

---

## 4) Utility (episode-level attribution, cheap and rebuildable)

No DecisionPoints, no arms.

### Utility signals

Utility is computed from **two observable streams**:

1. **Reuse**: card retrieved or shown again (Exposure counts).
2. **Success events**: anchored tool outcomes or explicit user confirmation.

### OutcomeEvent (episode-level)

**OutcomeEvent**: `(episode_id, outcome_type, evidence_refs...)`

Outcome types are a controlled vocabulary:

* `tool_success` (exit code ok, tests passed, request succeeded)
* `tool_failure` (tests failed, exception, revert)
* `user_confirmed_helpful`
* `user_corrected`

### Attribution rule (coarse, explicit)

For each episode:

* Define `eligible_cards = cards shown via auto_pack before the first success/failure outcome in that episode` (channel = auto_pack).
* For each eligible **Tactic**, update:

  * `wins += 1` if episode has tool_success or user_confirmed_helpful
  * `losses += 1` if episode has tool_failure or user_corrected

This is intentionally blunt. It is rebuildable because it uses Exposure + OutcomeEvent only.

### Guardrails against self-reinforcement (cheap)

* Only count episodes with at least one anchored tool outcome or explicit user confirmation.
* Cap per-episode credit: only top-K tactics (by retrieval score) in the auto_pack are eligible to receive credit, to prevent “everything gets credit.”

This keeps utility from inflating via mere presence.

---

## 5) Retrieval and packing (where v4 spends complexity)

### Ranking inputs

* scope match (repo > domain > global)
* lexical score (FTS)
* semantic score (cosine)
* truth weight for Fact/Preference/Constraint/Commitment
* utility weight for Tactic (from wins/losses + reuse)
* recency for relevance (applies to retrieval relevance, not truth)

### Packing (explicit slots + diversity)

Auto context pack is built by slot allocation, not top-K:

* Always include: top constraints/commitments in scope (with citations)
* Include: top negative results relevant to current errors/tools
* Include: a small number of tactics with highest utility *and* diverse topic_key
* Include: a small number of high-truth facts relevant to query

**Diversity rule**: max 1-2 cards per topic_key in the pack.

### Intent heuristics (simple, inspectable)

* If last tool output is failure: prioritize NegativeResults + repo facts + constraints
* If user prompt asks “how should / best way”: prioritize tactics + constraints
* If user prompt asks “what did I do last time”: prioritize facts + commitments and bypass auto-pack caps (because it is an explicit query)

---

## 6) Compaction (required)

CardStore is an index, so it must be compacted.

**Archive rules** (reversible):

* Archive cards with no exposures for a long window AND low utility/truth AND no disputes.
* Never delete; archived cards remain searchable but are excluded from auto_pack by default.

**Dedup sweep** runs periodically to merge duplicates.

---

## 7) Promotion (governance, not inference)

Promotion is rule-based:

* repo -> domain: tactic has wins in multiple repos OR explicit user_span: “this applies across my repos”
* domain -> global: only explicit user_span: “apply everywhere”

No statistical purity required in v4.

---

## 8) Staging gates for adding causal instrumentation later

Add DecisionPoints/arms only if v4 hits these gates:

1. **Retrieval stability**: pack precision is high by user confirmation; low complaint rate (“this is irrelevant”).
2. **Store boundedness**: budgets + archival keep active store size stable over time.
3. **Utility plateau**: tactics stop improving under episode-level attribution but there is reason to believe attribution is the limiter (e.g., many mixed-outcome episodes).
4. **Sufficient volume**: enough episodes per week that fine-grained attribution can converge.

Until then, causal machinery is postponed.

---

## Why this is “less” but keeps the compounding

It keeps the parts that prevent entropy (**evidence, budgets, packing, compaction, dispute/recheck**) and replaces the hardest-to-specify part (**decision credit assignment**) with a coarse, rebuildable loop that is usually good enough for a single user. When the system graduates from “useful memory” to “measurable learning system,” the log + exposure + outcome events are already the substrate required to add DecisionPoints without migration pain.