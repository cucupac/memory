# Improvement: Real Embedding Migration

Status: pinned
Priority: P0
Created: 2026-02-14

## Problem

Current semantic retrieval uses pseudo hash embeddings, which limits paraphrase
and synonym recall quality.

## Proposal

Adopt a real embedding model with versioned vectors and replay-safe migration:

- Keep `card_embeddings.embedding_model` version tags.
- Add migration command to backfill vectors for all existing cards.
- Preserve deterministic retrieval ordering with stable tie-break rules.

## Integration Points

- `/Users/adamcuculich/memory/memory_cli.py` `refresh_card_indices(...)`
- `/Users/adamcuculich/memory/memory_cli.py` `retrieve_cards(...)`
- `/Users/adamcuculich/memory/memory_cli.py` `migrate_embeddings(...)`

## Acceptance Criteria

- Semantic retrieval quality improves on paraphrase test set.
- Existing replay and idempotency checks still pass.
- Old vectors can coexist during staged migration.
- Rollback path exists via model-version switch.

