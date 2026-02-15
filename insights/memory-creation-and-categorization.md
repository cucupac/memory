# Memory Creation and Categorization

From design conversation 2026-02-14.

## Who creates memories

The LLM, at write time. Not a background process.

The LLM has full context — why it tried something, what surprised it, what the user meant. A background process reconstructing this from raw text loses what matters.

## Who categorizes

The LLM, at write time. It knows whether it observed a preference, an experience, or a structural fact. No heuristic classifier.

Write interface: LLM writes a formed memory + category tag + evidence refs for grounding.

## Structural memory is different

Not produced by a single episode. Built incrementally across sessions.

### Construction

- Starts empty. First episode, the LLM writes an initial coarse sketch.
- Each subsequent episode, the LLM reads the current structural map and either:
  - Does nothing. Nothing new learned.
  - Appends a detail.
  - Corrects something.
  - Refines granularity (breaks coarse node into finer sub-nodes).

### Coherence

Solved by read-before-write. The LLM always edits against the latest version. Sequential consistency, not parallel construction.

### Staleness

It will go stale. That's acceptable. A slightly stale bird's eye view is more useful than none. When the LLM works in an area that's changed, it corrects the map naturally through the normal write/dispute interfaces.

## Current design is flipped

Old: evidence in → heuristic classifier → card out.
New: LLM observes → LLM writes card with category → evidence attached for grounding.

The event log still records evidence refs. The card — the lossy abstraction — is authored by the LLM.
