# Improvement: Candidate Generation Classifier

Status: pinned
Priority: P0
Created: 2026-02-14

## Problem

Candidate extraction relies on keyword heuristics, which causes misclassification
and noisy admissions.

## Proposal

Replace heuristic kind detection with a constrained classifier stage:

- Input: evidence ref text + ref kind + episode scope metadata
- Output schema:
  - `kind` in `{preference,constraint,commitment,fact,tactic,negative_result}`
  - `statement` normalized text
  - `confidence` in `[0.0, 1.0]`
  - `why` short evidence-backed rationale
- Reject candidates below a configurable minimum confidence threshold.

## Integration Points

- `/Users/adamcuculich/memory/memory_cli.py` `generate_candidates(...)`
- `/Users/adamcuculich/memory/memory_cli.py` `consolidate_episode(...)`

## Acceptance Criteria

- Same inputs produce deterministic candidate ordering.
- Misclassified normative cards are reduced versus baseline.
- Ledger shows lower rejection rates for novelty/quality reasons.
- Feature flag allows fallback to legacy heuristic mode.

