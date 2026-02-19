# Memory Interface JSON Schemas v1

Status: ratified from approved proposal on 2026-02-18.

This card records the approved `read` / `write` / `update` interface semantics and the exact JSON schemas proposed.

## Operation Semantics

- `read`: retrieval only.
- `write`: creates immutable memory records.
- `update`: adjusts dynamic values (`truth`, `utility`) on existing memory IDs.
- "X changed in the codebase and invalidates Y" is a new memory and therefore uses `write` with `kind: "change"`.

## `memory.read.request`

```json
{
  "$id": "memory.read.request",
  "type": "object",
  "required": ["op", "repo_id", "mode", "query"],
  "properties": {
    "op": { "const": "read" },
    "repo_id": { "type": "string" },
    "mode": { "enum": ["ambient", "targeted"] },
    "query": { "type": "string", "minLength": 1 },
    "include_global": { "type": "boolean", "default": true },
    "kinds": {
      "type": "array",
      "items": { "enum": ["problem", "solution", "failed_tactic", "fact", "preference", "change"] },
      "uniqueItems": true
    },
    "limit": { "type": "integer", "minimum": 1, "maximum": 100, "default": 20 },
    "expand": {
      "type": "object",
      "properties": {
        "semantic_hops": { "type": "integer", "minimum": 0, "maximum": 3, "default": 2 },
        "include_problem_links": { "type": "boolean", "default": true },
        "include_update_links": { "type": "boolean", "default": true }
      }
    }
  }
}
```

## `memory.write.request`

```json
{
  "$id": "memory.write.request",
  "type": "object",
  "required": ["op", "repo_id", "memory"],
  "properties": {
    "op": { "const": "write" },
    "repo_id": { "type": "string" },
    "memory": {
      "type": "object",
      "required": ["text", "scope", "kind", "confidence"],
      "properties": {
        "text": { "type": "string", "minLength": 1 },
        "scope": { "enum": ["repo", "global"] },
        "kind": { "enum": ["problem", "solution", "failed_tactic", "fact", "preference", "change"] },
        "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
        "rationale": { "type": "string" },
        "links": {
          "type": "object",
          "properties": {
            "problem_id": { "type": "string" },
            "related_memory_ids": { "type": "array", "items": { "type": "string" } },
            "change_targets": { "type": "array", "items": { "type": "string" } }
          }
        },
        "evidence_refs": { "type": "array", "items": { "type": "string" } }
      }
    }
  }
}
```

## `memory.update.request`

```json
{
  "$id": "memory.update.request",
  "type": "object",
  "required": ["op", "repo_id", "memory_id", "mode", "updates"],
  "properties": {
    "op": { "const": "update" },
    "repo_id": { "type": "string" },
    "memory_id": { "type": "string" },
    "mode": { "enum": ["dry_run", "commit"] },
    "updates": {
      "type": "object",
      "properties": {
        "truth": {
          "type": "object",
          "required": ["target", "confidence", "rationale", "evidence_refs"],
          "properties": {
            "target": { "type": "number", "minimum": 0, "maximum": 1 },
            "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
            "rationale": { "type": "string" },
            "evidence_refs": { "type": "array", "items": { "type": "string" }, "minItems": 1 }
          }
        },
        "utility": {
          "type": "object",
          "required": ["target", "confidence", "rationale"],
          "properties": {
            "target": { "type": "number", "minimum": 0, "maximum": 1 },
            "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
            "rationale": { "type": "string" },
            "context_problem_id": { "type": "string" },
            "evidence_refs": { "type": "array", "items": { "type": "string" } }
          }
        }
      },
      "anyOf": [{ "required": ["truth"] }, { "required": ["utility"] }]
    }
  }
}
```

## Validation Placement

A validation layer sits directly under the interface and performs:
- Schema validation: shape, types, enums, required fields.
- Semantic validation: domain constraints (for example, evidence required for truth updates).
