# Plan C Implementation Plan (Phases 1-4 Executable Spec)

This file implements phases 1-4 from `/Users/adamcuculich/memory/architectures/plan_c.md` as a concrete build specification.

## Baseline decisions

- Canonical interaction truth: append-only `episodes`.
- Canonical memory-system truth: append-only `memory_events` from day 1.
- All mutable tables are projections/reducers from `episodes + memory_events`.
- Determinism first: stable ordering, explicit reason codes, idempotent writes.
- Learning updates only from observable evidence (`tool_output`, `doc_span`, `user_span`).
- Local-first deployment: SQLite + local artifact files + CLI.

## Default policy constants (v1)

### Evidence invariants and supersession matrix

| Card kind | Required evidence refs | Supersession/deprecation rule |
| --- | --- | --- |
| `preference` | >=1 `user_span` explicit preference statement | Only superseded by later explicit `user_span` |
| `constraint` | >=1 `user_span` explicit constraint statement | Only superseded by later explicit `user_span` |
| `commitment` | >=1 `user_span` explicit commitment statement | Superseded by later explicit `user_span`; may be marked completed from tool evidence |
| `fact` | >=1 anchored ref (`tool_output` or `doc_span` or `user_span`) | Contradictions require anchored evidence; disputed facts can move to `needs_recheck` |
| `tactic` | >=1 `tool_output` or `doc_span` demonstrating procedure/behavior | Deprecated by anchored failure evidence or explicit user correction |
| `negative_result` | >=1 `tool_output` failure evidence | Deprecated when later anchored evidence shows issue resolved |

### Budget defaults (hard caps) by `(scope_tier, kind)`

| Scope | preference | constraint | commitment | fact | tactic | negative_result |
| --- | --- | --- | --- | --- | --- | --- |
| `repo` | 80 | 120 | 120 | 300 | 120 | 120 |
| `domain` | 40 | 60 | 60 | 180 | 80 | 80 |
| `global` | 20 | 30 | 30 | 100 | 40 | 40 |

Additional write-path limits:
- Soft per-episode admitted-card cap: `12`
- Per-episode per-kind caps: `fact=4`, `tactic=2`, `negative_result=2`, `preference=2`, `constraint=1`, `commitment=1`

### Retrieval and packing defaults

- Auto-pack total card cap: `8`
- Slot caps (default context): constraints+commitments `3`, negative_results `2`, tactics `2`, facts `3`
- Topic diversity cap: max `2` cards per `topic_key`
- Failure context override: reserve at least `1` negative_result slot when a recent tool failure exists

### Dispute and attribution defaults

- Dispute evidence weights: `tool_output=1.0`, `doc_span=0.7`, `user_span=0.4`
- `needs_recheck` threshold by scope: `repo>=2.0`, `domain>=3.0`, `global>=4.0`
- Utility credit cap: top-`2` tactics per episode can receive win/loss updates

## Phase 1: Canonical Log + Event Substrate (implemented spec)

### Schema

```sql
CREATE TABLE IF NOT EXISTS episodes (
  episode_id TEXT PRIMARY KEY,
  user_text TEXT NOT NULL,
  assistant_text TEXT NOT NULL,
  model_name TEXT,
  metadata_json TEXT NOT NULL DEFAULT '{}',
  payload_hash TEXT NOT NULL,
  started_at TEXT NOT NULL,
  ended_at TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS artifacts (
  artifact_id TEXT PRIMARY KEY,
  episode_id TEXT NOT NULL REFERENCES episodes(episode_id),
  artifact_kind TEXT NOT NULL CHECK (artifact_kind IN ('tool_output', 'doc')),
  content_path TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  mime_type TEXT,
  metadata_json TEXT NOT NULL DEFAULT '{}',
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS evidence_refs (
  evidence_ref_id TEXT PRIMARY KEY,
  episode_id TEXT NOT NULL REFERENCES episodes(episode_id),
  artifact_id TEXT REFERENCES artifacts(artifact_id),
  ref_kind TEXT NOT NULL CHECK (ref_kind IN ('user_span', 'tool_output', 'doc_span')),
  target_id TEXT NOT NULL,
  start_offset INTEGER,
  end_offset INTEGER,
  line_start INTEGER,
  line_end INTEGER,
  ref_hash TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS memory_events (
  event_id INTEGER PRIMARY KEY AUTOINCREMENT,
  episode_id TEXT NOT NULL REFERENCES episodes(episode_id),
  seq_no INTEGER NOT NULL,
  event_type TEXT NOT NULL,
  payload_json TEXT NOT NULL,
  payload_hash TEXT NOT NULL,
  idempotency_key TEXT NOT NULL UNIQUE,
  producer TEXT NOT NULL,
  rule_version TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE (episode_id, seq_no)
);

CREATE INDEX IF NOT EXISTS idx_memory_events_episode ON memory_events (episode_id, seq_no);
CREATE INDEX IF NOT EXISTS idx_memory_events_type ON memory_events (event_type, created_at);
```

### Required event types in phase 1

- `episode_recorded`
- `artifact_recorded`
- `evidence_ref_recorded`
- `consolidation_triggered`
- `exposure_recorded`
- `outcome_recorded`

Event payloads are JSON objects and must include:
- `schema_version`
- deterministic primary IDs used by reducers
- optional `reason_code` when decisioning is involved

### Append API contract

- `append_event(episode_id, event_type, payload_json, idempotency_key, producer, rule_version)`
- Sequence allocation: `seq_no = 1 + COALESCE(MAX(seq_no), 0)` per episode in the same transaction.
- Idempotency rule: duplicate `idempotency_key` must return existing `(event_id, episode_id, seq_no)` and never create a second row.
- Payload hash: `sha256(canonical_json(payload_json))`.

### CLI surface

- `memory record-episode --input <episode.json>`
- `memory append-event --episode <id> --type <event_type> --payload <payload.json> --idempotency-key <k>`
- `memory replay --from-event-id <id>`
- `memory export --episode <id> --format jsonl`

### Phase 1 acceptance tests

- Retried `append-event` with same idempotency key keeps exactly one row in `memory_events`.
- Replaying all events after truncating reducers reproduces byte-identical projections.
- Recording an episode persists `episodes`, `artifacts`, `evidence_refs`, and at least one `episode_recorded` event.

## Phase 2: Card Reducers + Deterministic Consolidation (implemented spec)

### Projection schema

```sql
CREATE TABLE IF NOT EXISTS cards (
  card_id TEXT PRIMARY KEY,
  kind TEXT NOT NULL CHECK (kind IN ('preference', 'constraint', 'commitment', 'fact', 'tactic', 'negative_result')),
  statement TEXT NOT NULL,
  scope_tier TEXT NOT NULL CHECK (scope_tier IN ('repo', 'domain', 'global')),
  scope_id TEXT NOT NULL,
  topic_key TEXT NOT NULL,
  tags_json TEXT NOT NULL DEFAULT '[]',
  status TEXT NOT NULL CHECK (status IN ('active', 'needs_recheck', 'deprecated', 'archived')),
  supersedes_card_id TEXT REFERENCES cards(card_id),
  created_event_id INTEGER NOT NULL,
  updated_event_id INTEGER NOT NULL,
  archived_at TEXT
);

CREATE TABLE IF NOT EXISTS card_evidence_refs (
  card_id TEXT NOT NULL REFERENCES cards(card_id),
  evidence_ref_id TEXT NOT NULL REFERENCES evidence_refs(evidence_ref_id),
  PRIMARY KEY (card_id, evidence_ref_id)
);

CREATE TABLE IF NOT EXISTS consolidation_ledger (
  episode_id TEXT PRIMARY KEY,
  proposed_count INTEGER NOT NULL,
  admitted_count INTEGER NOT NULL,
  rejected_count INTEGER NOT NULL,
  merged_count INTEGER NOT NULL,
  superseded_count INTEGER NOT NULL,
  archived_count INTEGER NOT NULL,
  reason_breakdown_json TEXT NOT NULL,
  computed_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Consolidation event catalog

- `candidate_proposed`
- `card_admitted`
- `card_rejected`
- `card_merged`
- `card_superseded`
- `card_archived`

Required rejection reason codes:
- `missing_required_evidence`
- `novelty_below_threshold`
- `scope_kind_budget_exceeded`
- `episode_kind_cap_exceeded`
- `episode_soft_cap_exceeded`
- `duplicate_of_existing_card`

### Deterministic consolidation algorithm

1. Load episode artifacts and evidence refs in stable order by `(created_at, evidence_ref_id)`.
2. Generate candidates sorted by `(kind_priority, normalized_statement, scope_tier, scope_id)`.
3. Validate evidence invariants per kind.
4. Compute novelty:
   - lexical similarity (Jaccard of normalized token set)
   - semantic similarity (cosine with embedding model pinned by version)
5. Reject candidate when `max_similarity >= duplicate_threshold`:
   - default duplicate threshold: cosine `>= 0.92` and lexical `>= 0.80`
6. Apply budget checks in strict order:
   - per-episode per-kind cap
   - per-episode soft cap
   - per `(scope_tier, kind)` hard cap
7. For admitted candidate, deterministically choose action:
   - new card
   - merge evidence into existing card
   - supersede existing card (normative kinds by explicit `user_span` only)
8. Emit event for every decision with reason code and threshold values.
9. Rebuild `consolidation_ledger` from events, not from in-memory counters.

Kind priority for stable sorting:
- `constraint`, `commitment`, `preference`, `negative_result`, `tactic`, `fact`

### Entropy controls

- Write-time near-duplicate suppression during consolidation.
- Daily dedup job:
  - scan active cards by `(kind, scope_tier, scope_id)`
  - merge deterministic winner by `(highest evidence count, newest updated_event_id, lowest card_id)`
  - emit `card_merged` for every merged card.

### CLI surface

- `memory consolidate --episode <id>`
- `memory ledger --episode <id>`
- `memory dedup --daily`

### Phase 2 acceptance tests

- Consolidation auto-runs after `record-episode`.
- Every rejected candidate has a reason code and relevant threshold fields.
- Re-running consolidation on the same event stream yields no additional admitted cards.

## Phase 3: Retrieval, Deterministic Packing, Exposure, Explainability (implemented spec)

### Retrieval schema and indices

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS cards_fts USING fts5(
  card_id UNINDEXED,
  statement,
  topic_key,
  tags,
  content='',
  tokenize='porter unicode61'
);

CREATE TABLE IF NOT EXISTS card_embeddings (
  card_id TEXT PRIMARY KEY REFERENCES cards(card_id),
  embedding_model TEXT NOT NULL,
  embedding_vector BLOB NOT NULL,
  updated_event_id INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS pack_snapshots (
  pack_id TEXT PRIMARY KEY,
  episode_id TEXT NOT NULL REFERENCES episodes(episode_id),
  channel TEXT NOT NULL CHECK (channel IN ('auto_pack', 'search', 'explicit_read', 'check')),
  query_text TEXT,
  policy_version TEXT NOT NULL,
  ranked_candidates_json TEXT NOT NULL,
  selected_cards_json TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS exposures (
  exposure_id TEXT PRIMARY KEY,
  episode_id TEXT NOT NULL REFERENCES episodes(episode_id),
  pack_id TEXT REFERENCES pack_snapshots(pack_id),
  card_id TEXT NOT NULL REFERENCES cards(card_id),
  channel TEXT NOT NULL CHECK (channel IN ('auto_pack', 'search', 'explicit_read', 'check')),
  rank_position INTEGER,
  score_total REAL,
  source_event_id INTEGER NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Retrieval ranking model

Inputs:
- scope match (`repo > domain > global`)
- lexical score (FTS/BM25 normalized)
- semantic score (cosine)
- kind prior
- truth/lifecycle weight
- utility weight (tactic only; phase 4B)
- recency relevance weight

Deterministic tie-break order:
- `score_total DESC, kind_priority ASC, updated_event_id DESC, card_id ASC`

`needs_recheck` cards:
- auto-pack downrank multiplier: `0.35`
- explicit search/read/check: no exclusion

### Pack builder algorithm

1. Build ranked candidates and persist full ranked list in `pack_snapshots`.
2. Allocate slots in order:
   - constraints+commitments (up to 3)
   - negative results (up to 2)
   - tactics (up to 2)
   - facts (up to 3)
3. Enforce total cap `8`.
4. Enforce topic diversity: max `2` cards per `topic_key`.
5. Failure override: if recent tool failure exists, reserve >=1 negative result slot.
6. Persist selected cards and score components.
7. Emit `exposure_recorded` event and insert per-card `exposures` rows.

### Explainability CLI

- `memory explain-pack --episode <id>`
  - shows candidate components, thresholds, slot decisions, diversity drops, citations
- `memory explain-consolidation --episode <id>`
  - shows proposal/admit/reject/merge/supersede chain with reason codes

### Archive hygiene pass

- Archive eligibility check runs before ranking:
  - status is `active`
  - no disputes above threshold
  - not exposed in recent window
  - low utility/truth score
- Archiving is event-backed via `card_archived`.

### Phase 3 acceptance tests

- Auto-pack contains <=8 cards and honors slot and diversity caps.
- Explicit `search` returns archived and `needs_recheck` cards.
- `explain-pack` reproduces exactly the selected set from `pack_snapshots`.

## Phase 4: Checkpointed Learning Loop (4A + 4B implemented spec)

### Phase 4A: Truth, dispute, lifecycle

#### Additional schema

```sql
CREATE TABLE IF NOT EXISTS disputes (
  dispute_id TEXT PRIMARY KEY,
  card_id TEXT NOT NULL REFERENCES cards(card_id),
  evidence_ref_id TEXT NOT NULL REFERENCES evidence_refs(evidence_ref_id),
  weight REAL NOT NULL,
  event_id INTEGER NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS card_status_history (
  card_id TEXT NOT NULL REFERENCES cards(card_id),
  event_id INTEGER NOT NULL,
  from_status TEXT NOT NULL,
  to_status TEXT NOT NULL,
  reason_code TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (card_id, event_id)
);
```

#### Lifecycle event catalog

- `dispute_recorded`
- `card_status_changed`
- `card_archived`
- `card_deprecated`

#### Dispute and status transition rules

- Contradiction weights:
  - `tool_output=1.0`
  - `doc_span=0.7`
  - `user_span=0.4`
- For each fact card, `dispute_mass = SUM(active dispute weights)`.
- Transition to `needs_recheck` when:
  - repo scope: `dispute_mass >= 2.0`
  - domain scope: `dispute_mass >= 3.0`
  - global scope: `dispute_mass >= 4.0`
- Transition to `deprecated` only with anchored resolution/failure evidence.
- No hard deletion; lifecycle is status-only and event-backed.

#### Promotion rules

- `repo -> domain` if:
  - explicit `user_span` says broader applicability, or
  - tactic has positive outcomes in at least 2 repos.
- `domain -> global` only via explicit `user_span`.

Every promotion emits `card_status_changed` or `card_superseded` with reason code.

### Phase 4B: Utility attribution and tactic learning

#### Additional schema

```sql
CREATE TABLE IF NOT EXISTS utility_stats (
  card_id TEXT PRIMARY KEY REFERENCES cards(card_id),
  wins INTEGER NOT NULL DEFAULT 0,
  losses INTEGER NOT NULL DEFAULT 0,
  reuse INTEGER NOT NULL DEFAULT 0,
  updated_event_id INTEGER NOT NULL
);
```

#### Outcome event contract

`outcome_recorded` payload:
- `outcome_type` in:
  - `tool_success`
  - `tool_failure`
  - `user_confirmed_helpful`
  - `user_corrected`
- `evidence_ref_ids` (must include anchored refs)
- optional `tool_name`, `metadata_json`

#### Deterministic attribution algorithm

1. Identify first terminal outcome `seq_no` in episode.
2. Build eligible tactics:
   - card kind = `tactic`
   - shown in `auto_pack`
   - exposure event `seq_no < first_terminal_outcome_seq_no`
3. Order eligible tactics by:
   - `rank_position ASC`
   - `score_total DESC`
   - `card_id ASC`
4. Credit only top `2` tactics.
5. Update utility:
   - if episode has `tool_success` or `user_confirmed_helpful`, increment `wins`
   - if episode has `tool_failure` or `user_corrected`, increment `losses`
   - increment `reuse` from exposures (all channels).
6. Persist all updates as reducer outputs from `outcome_recorded` and `exposure_recorded` events.

### Phase 4 acceptance tests

- Facts with high dispute mass are downranked in auto-pack without deletion.
- Lifecycle transitions replay identically from event log.
- Attribution from replay matches live attribution for the same episode.
- Repeated positive outcomes increase tactic rank over time.

## Phase 5: Hardening, Rebuildability, and Production Gates (implemented spec)

### Operator CLI and dashboards

- `memory status --days <n>`
  - store health checks
  - counts by kind/status/scope
  - consolidation acceptance/rejection trend
  - retrieval precision proxy + correction-rate trend
- `memory gates --days <n>`
  - evaluates readiness gates for adding causal instrumentation

### One-command full rebuild

- `memory full-rebuild --verify-stability`
  - rebuilds all projections from `memory_events`
  - verifies deterministic digest stability on repeated rebuilds

### Failure handling and recovery

- `memory recover`
  - repairs missing canonical events (`episode_recorded`, `artifact_recorded`, `evidence_ref_recorded`, `consolidation_triggered`)
  - optionally runs missing consolidations
- `memory verify-idempotency --sample-events <n>`
  - validates replay determinism and idempotent append behavior

### Embedding/version migration

- `memory migrate-embeddings --to-model <name> [--from-model <name>] [--dim <n>]`
  - recomputes embedding vectors with model-tagged versioning

### Phase 5 acceptance tests

- `status` returns health + trend metrics without reducer drift.
- `full-rebuild --verify-stability` reports identical digests across repeated rebuilds.
- `verify-idempotency` reports no inserts on idempotency-key retry.
- `gates` returns deterministic pass/fail with explicit threshold outputs.
