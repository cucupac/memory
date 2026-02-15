# Improvement: Git-Diff Staleness Invalidation

Status: pinned
Priority: P1
Created: 2026-02-14

## Problem

Repo-scoped memories can become stale after code changes, but staleness updates
are currently manual.

## Proposal

Add automatic invalidation from codebase changes:

- Detect touched files from local git diff per episode.
- Link cards to code evidence where possible.
- If changed files intersect linked evidence paths, emit:
  - `card_status_changed` -> `needs_recheck`
  - reason code: `repo_change_possible_staleness`

## Integration Points

- New ingestion metadata on evidence refs for file paths/hashes.
- New invalidation pass before pack build.
- Existing lifecycle reducers handle status updates.

## Acceptance Criteria

- Changed relevant files reduce stale-card auto-pack usage.
- Unrelated changes do not mass-downgrade cards.
- All invalidations are event-backed and replayable.
- `status` report includes invalidation counts/trend.

