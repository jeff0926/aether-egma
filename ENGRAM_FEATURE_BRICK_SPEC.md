# Feature Brick Spec: `engram.py`
**Project:** AETHER  
**Module:** `engram.py`  
**Version:** v1.0  
**Status:** Ready for build  
**Date:** 2026-03-26  
**Owner:** 864zeros LLC / Tom (Temporal Orchestration Mechanism)

---

## 1. Purpose

`engram.py` is the **Subgraph Persistence / Working Memory layer** for AETHER capsules.

An engram (neuroscience) is the physical memory trace a memory leaves in the brain. This module manages the equivalent for an AETHER agent: which KG nodes were "activated" during a session, how strongly, and what persists into the next.

**Core job:** At session end, serialize the active subgraph as a JSON-LD manifest (`engram.jsonld`). At session start, reload it and warm the agent's context. Pass node URIs — not raw text — across turns and sessions.

---

## 2. Brick Boundary

### What `engram.py` OWNS
- Node salience scoring (direct mention / inferred context / decay)
- Active subgraph extraction (max-hop traversal from salient nodes)
- Engram Manifest serialization and deserialization (`engram.jsonld`)
- Conflict detection trigger (detect contradiction → call `kg.mark_deprecated`)
- Decay computation (reduce salience of unaccessed nodes over time)

### What `engram.py` does NOT own
- KG persistence → `kg.py` owns `save_kg` / `load_kg`
- Triple validation → `aec.py` owns `verify`
- Self-education queue → `education.py` owns `queue_failure`
- Capsule loading → `aether.py` owns `Capsule`

### Integration points
| Calls | Purpose |
|-------|---------|
| `kg.load_kg()` | Load KG to traverse |
| `kg.get_nodes()` | Enumerate all nodes |
| `kg.mark_deprecated()` | Tombstone contradicted node |
| `kg.mark_updated()` | Update node on non-contradicting refresh |

---

## 3. KG Schema Additions Required

Two new fields must be added to `kg.py`'s `add_knowledge()` node structure.  
These are **optional fields** — existing nodes without them are treated as `last_accessed = acquired_date`, `access_count = 0`.

```python
# Add to node creation block in kg.py → add_knowledge()
"aether:last_accessed": datetime.now().isoformat(),
"aether:access_count": 0,
```

Add a helper to `kg.py`:

```python
def touch_node(kg: dict, node_id: str) -> dict:
    """Increment access_count and update last_accessed on a node."""
    for node in get_nodes(kg):
        if node.get("@id") == node_id:
            node["aether:last_accessed"] = datetime.now().isoformat()
            node["aether:access_count"] = node.get("aether:access_count", 0) + 1
            break
    return kg
```

---

## 4. `definition.json` Schema Addition

Add an optional `"engram"` block to `definition.json`. If absent, use defaults below.

```json
"engram": {
  "max_hop": 2,
  "max_nodes": 20,
  "decay_rate": 0.15,
  "salience_threshold": 0.2
}
```

| Field | Default | Meaning |
|-------|---------|---------|
| `max_hop` | `2` | Max degrees of separation from salient nodes |
| `max_nodes` | `20` | Hard cap on nodes in the active subgraph |
| `decay_rate` | `0.15` | Salience reduction per turn for unaccessed nodes |
| `salience_threshold` | `0.2` | Nodes below this are excluded from manifest |

---

## 5. Public API

### 5.1 `score_salience(nodes, mentioned_ids, turn_count) → dict`

Compute salience scores for all nodes.

```python
def score_salience(
    nodes: list[dict],
    mentioned_ids: list[str],   # node @ids directly mentioned this turn
    turn_count: int = 0,        # how many turns since each node was last accessed
) -> dict:                      # returns {node_id: float 0.0–1.0}
```

**Scoring rules:**
- Direct mention (`@id` in `mentioned_ids`): `1.0`
- Inferred (neighbor of direct mention within 1 hop): `0.6`
- Historical (accessed before, not this turn): `max(0.0, prior_score - decay_rate * turns_since_access)`
- Never accessed: `0.0`

Deprecated nodes are always scored `0.0` regardless of mention.

---

### 5.2 `extract_subgraph(kg, salient_ids, max_hop, max_nodes) → list[dict]`

Return the active subgraph — nodes reachable from salient nodes within `max_hop` degrees.

```python
def extract_subgraph(
    kg: dict,
    salient_ids: list[str],   # node @ids with salience above threshold
    max_hop: int = 2,
    max_nodes: int = 20,
) -> list[dict]:              # returns list of node dicts, ordered by salience desc
```

**Traversal rules:**
- BFS from each salient node
- Follow any field whose value matches a known `@id` in the KG (treat as edge)
- Stop at `max_hop` depth
- If result exceeds `max_nodes`, keep highest-salience nodes
- Never include deprecated nodes

---

### 5.3 `build_manifest(capsule_id, subgraph, salience_scores, turn_id) → dict`

Serialize the active subgraph as a JSON-LD Engram Manifest.

```python
def build_manifest(
    capsule_id: str,           # capsule name / @id
    subgraph: list[dict],      # output of extract_subgraph
    salience_scores: dict,     # {node_id: float} from score_salience
    turn_id: str = "",         # optional turn identifier
) -> dict:                     # returns JSON-LD manifest dict
```

**Output schema:**

```json
{
  "@context": {
    "aether": "http://aether.dev/ontology#",
    "engram": "http://aether.dev/engram#"
  },
  "@type": "engram:Manifest",
  "engram:capsule_id": "scholar-buffett",
  "engram:turn_id": "t-042",
  "engram:created": "2026-03-26T00:00:00Z",
  "engram:schema_version": "1.0",
  "engram:source_capsule": "scholar-buffett",
  "engram:active_nodes": [
    {
      "@id": "aether:core/warren_buffett",
      "engram:salience": 0.9,
      "engram:hop_depth": 0
    }
  ],
  "engram:node_count": 1,
  "engram:max_hop_used": 2
}
```

**A2A readiness:** `engram:source_capsule` is always populated. When passed to another capsule, the receiver knows the provenance. No additional schema changes needed for A2A v2.

---

### 5.4 `save_manifest(manifest, capsule_path) → Path`

Write `engram.jsonld` into the capsule folder as optional sidecar.

```python
def save_manifest(
    manifest: dict,
    capsule_path: str | Path,
) -> Path:                    # returns path to written file
```

Writes to `{capsule_path}/engram.jsonld`. Overwrites if exists (last session wins).

---

### 5.5 `load_manifest(capsule_path) → dict | None`

Load engram from capsule folder. Returns `None` if no prior engram exists (cold start).

```python
def load_manifest(capsule_path: str | Path) -> dict | None:
```

---

### 5.6 `detect_conflict(kg, statement, active_node_ids) → dict`

Check whether a new statement contradicts any currently active KG node.

```python
def detect_conflict(
    kg: dict,
    statement: str,            # new user/agent statement to check
    active_node_ids: list[str] # currently active nodes to check against
) -> dict:
# returns:
# {
#   "conflict": bool,
#   "conflicted_node_id": str | None,
#   "reason": str
# }
```

**Conflict detection rules (v1 — lightweight):**
- Tokenize the statement
- For each active node, check if the node's label appears AND the statement contains a contradicting verb pattern:
  - `sold`, `no longer`, `deleted`, `removed`, `cancelled`, `changed`, `not anymore`
- If match: return conflict = True, conflicted_node_id = the matched node
- This is intentionally simple for v1. NLI upgrade is deferred.

**On conflict detected, caller is responsible for:**
```python
kg.mark_deprecated(kg, conflicted_node_id, reason="engram_conflict: {statement[:80]}")
```

---

### 5.7 `warm_context(capsule_path, kg) → dict`

Load prior engram and return the active node list ready for context injection.  
This is the **session start** entrypoint.

```python
def warm_context(
    capsule_path: str | Path,
    kg: dict,
) -> dict:
# returns:
# {
#   "warm": bool,                  # False if no prior engram (cold start)
#   "active_nodes": list[dict],    # full node dicts from KG matching manifest
#   "missing_ids": list[str],      # manifest IDs that no longer exist in KG
#   "manifest": dict | None        # raw manifest, or None
# }
```

`missing_ids` indicates nodes that were in the prior engram but have since been deprecated or removed from the KG. Log these — they indicate KG churn between sessions.

---

### 5.8 `commit_session(capsule_id, capsule_path, kg, mentioned_ids, turn_id) → dict`

**Session end entrypoint.** Run full engram pipeline and write sidecar.

```python
def commit_session(
    capsule_id: str,
    capsule_path: str | Path,
    kg: dict,
    mentioned_ids: list[str],  # node @ids touched this session
    turn_id: str = "",
    config: dict = None,       # engram block from definition.json; uses defaults if None
) -> dict:
# returns:
# {
#   "manifest_path": str,
#   "node_count": int,
#   "manifest": dict
# }
```

Internally calls: `score_salience` → `extract_subgraph` → `build_manifest` → `save_manifest`.

---

## 6. File Produced

| File | Location | Type | When Created |
|------|----------|------|--------------|
| `engram.jsonld` | `{capsule_path}/engram.jsonld` | Optional sidecar | On `commit_session()` |

The 5-file capsule contract is **unchanged**. `engram.jsonld` is present only when a session has been committed. Absent = cold start. The stamper should include it in zip archives when present but never require it.

---

## 7. Stamper Integration

Add to `stamper.py`'s archive/copy logic:

```python
OPTIONAL_CAPSULE_FILES = ["engram.jsonld"]
```

When stamping a capsule, include `engram.jsonld` if it exists. Never fail if absent.

---

## 8. What Is NOT in v1

| Deferred | Reason |
|----------|--------|
| A2A receiver logic (loading another capsule's engram) | Needs A2A protocol; schema is ready |
| NLI-based conflict detection | v1 uses lightweight verb pattern matching |
| Multi-session decay accumulation | v1 resets salience each session |
| Engram merge (two capsules sharing a manifest) | Future shared memory model |
| Engram diff / changelog between sessions | Useful, deferred |

---

## 9. Test Coverage Required

```
test_score_salience_direct_mention       → score == 1.0
test_score_salience_inferred             → score == 0.6
test_score_salience_decay                → score decreases with turn_count
test_score_salience_deprecated_excluded  → deprecated nodes always 0.0
test_extract_subgraph_max_hop            → nodes beyond max_hop excluded
test_extract_subgraph_max_nodes          → result never exceeds max_nodes
test_build_manifest_schema               → all required fields present
test_save_load_manifest_roundtrip        → save then load returns equivalent manifest
test_warm_context_cold_start             → warm=False when no engram.jsonld
test_warm_context_warm_start             → warm=True, active_nodes populated
test_warm_context_missing_ids            → missing_ids populated for stale nodes
test_detect_conflict_verb_pattern        → "I sold the car" conflicts "car" node
test_detect_conflict_no_false_positive   → "I bought a car" does not conflict
test_commit_session_writes_file          → engram.jsonld created in capsule_path
```

---

## 10. Claude Code Prompt

> Read this spec fully before writing a single line of code.
>
> Build `engram.py` in `C:\Users\I820965\dev\aether\` using the public API in Section 5 exactly as specified. Standard library + datetime only — no new dependencies.
>
> Also apply the KG schema additions from Section 3 to `kg.py` (`touch_node` helper + two new fields in `add_knowledge`).
>
> Also apply the stamper change from Section 7 to `stamper.py`.
>
> Write tests for all 13 cases in Section 9 into `tests/test_engram.py`.
>
> Do not modify `aec.py`, `education.py`, or `aether.py`. `engram.py` calls into `kg.py` only.
>
> When done, run `python engram.py` with a self-test block (like the pattern in `kg.py`) and confirm all tests pass.

---

*Spec authored by Tom (Temporal Orchestration Mechanism) — AETHER Session 2026-03-26*  
*Locked. Do not modify without explicit Jeff approval.*
