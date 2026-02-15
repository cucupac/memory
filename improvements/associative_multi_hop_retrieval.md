# Improvement: Associative Multi-Hop Retrieval

Status: pinned
Created: 2026-02-14

## Idea

Add optional associative retrieval on top of direct query retrieval:

1. Prompt query -> retrieve top memory candidates (hop 0).
2. For each selected memory, retrieve semantically nearest memories (hop 1).
3. Repeat up to a capped depth (default `3` hops).

This allows useful context chains such as:

`prompt -> memory_1 -> memory_2 -> memory_3`

## Interface With Current Code

Primary integration points:

- `retrieve_cards(...)` in `/Users/adamcuculich/memory/memory_cli.py`
- `build_pack(...)` in `/Users/adamcuculich/memory/memory_cli.py`
- `card_embeddings` table + `cosine_from_vectors(...)` in `/Users/adamcuculich/memory/memory_cli.py`

Proposed path:

1. Keep current retrieval as seed ranking.
2. Add an expansion step that walks neighbors by embedding cosine.
3. Merge direct and associative scores.
4. Pass merged candidates into existing slot/diversity pack logic unchanged.

## Suggested v1 Algorithm

- `hop_limit = 3`
- `beam_width = 2` per node
- `min_cosine = 0.72`
- `hop_decay = 0.80 ** hop`
- Track `visited_card_ids` to prevent loops
- Prefer same `scope_tier/scope_id` before cross-scope neighbors
- Keep existing status filtering:
  - `auto_pack`: only `active`, `needs_recheck`
  - explicit modes may include archived/deprecated

Score blend example:

- `final_score = max(direct_score, assoc_score)`
- where `assoc_score` uses:
  - cosine similarity to parent card
  - parent score
  - hop decay
  - scope/status penalties

## Guardrails

- Deterministic ordering/tie-breaks must remain stable.
- Preserve existing pack constraints:
  - slot caps
  - topic diversity cap
  - `needs_recheck` downrank
- Limit explosion with strict hop/beam caps.
- Consider excluding `hop_depth > 0` tactics from utility credit, or downweighting them.

## Acceptance Criteria

- Optional feature flag (off by default), for example:
  - `pack --assoc-hops 3`
  - `search --assoc-hops 3`
- With flag off, outputs remain byte-identical to current behavior.
- With flag on, results are deterministic for the same DB state/query.
- Runtime remains bounded (no unbounded graph walk).

