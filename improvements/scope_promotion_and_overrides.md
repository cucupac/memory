# Improvement: Scope Promotion and Overrides

Status: pinned
Priority: P1
Created: 2026-02-14

## Problem

Scope tiers exist (`repo`, `domain`, `global`) but automatic promotion and user
override controls are not implemented.

## Proposal

Implement rule-based promotion plus explicit user controls:

- Auto promotion:
  - `repo -> domain` when corroborated across repos or explicit user instruction
  - `domain -> global` only on explicit user instruction
- Manual commands:
  - `promote --card-id ... --to-scope domain|global`
  - `demote --card-id ... --to-scope repo|domain`
  - `pin --card-id ...` and `unpin --card-id ...`

## Integration Points

- Card mutation events remain append-only and reducer-applied.
- Retrieval scope scoring consumes promoted scope fields.
- Consolidation ledger includes promotion/demotion reason codes.

## Acceptance Criteria

- Promotions are deterministic and replayable.
- Manual overrides win over auto rules when conflicts occur.
- Explain endpoints show why scope changed.
- Cross-repo useful tactics become discoverable at domain/global tiers.

