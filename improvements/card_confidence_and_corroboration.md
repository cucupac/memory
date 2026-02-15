# Improvement: Card Confidence and Corroboration

Status: pinned
Priority: P0
Created: 2026-02-14

## Problem

There is no single confidence scalar for cards. Retrieval currently infers truth
from status and utility proxies spread across projections.

## Proposal

Add a deterministic confidence model per card:

- Inputs:
  - evidence count
  - evidence kind diversity
  - independent episode count
  - independent artifact/source hash count
  - dispute mass and contradiction count
- Output:
  - `confidence_score` in `[0.0, 1.0]`
  - `confidence_components_json` for explainability

## Integration Points

- New projection table: `card_confidence`
- Recomputed in reducers after:
  - `card_admitted`
  - `card_merged`
  - `dispute_recorded`
  - `card_status_changed`
- Retrieval blend in `/Users/adamcuculich/memory/memory_cli.py` `retrieve_cards(...)`

## Acceptance Criteria

- Confidence rises with independent corroboration.
- Confidence drops when dispute mass rises.
- Pack ranking uses confidence for fact-like kinds.
- `explain-pack` includes confidence and component trace.

