# Claude Code Diary Entry
**Date:** 2026-03-26 23:07 (Updated: 2026-03-27 10:45)
**Session:** Engram Feature Brick Implementation
**Author:** Claude Opus 4.5
**Repositories:**
- **aether** (code): https://github.com/jeff0926/aether.git
- **aether-egma** (docs/specs): https://github.com/jeff0926/aether-egma.git

---

## Repository Structure

| Repo | Purpose | Contents |
|------|---------|----------|
| `aether/` | Code | `engram.py`, `kg.py`, `stamper.py`, `tests/test_engram.py`, all core `.py` files |
| `aether-egma/` | Docs & Specs | `ENGRAM_FEATURE_BRICK_SPEC.md`, `AETHER_ENGRAM_PROJECT_OVERVIEW.md`, diary, deferred-topics, test-suite docs |

---

## Executive Summary

Implemented `engram.py` - the **Subgraph Persistence / Working Memory layer** for AETHER capsules. This module manages which KG nodes were "activated" during a session, their salience scores, and what persists to the next session via `engram.jsonld` sidecar files.

---

## What Is AETHER?

AETHER is an agent framework built around **capsules** - self-contained knowledge packages with 5 required files:
- `{id}-manifest.json` - Capsule metadata (id, version, timestamps)
- `{id}-definition.json` - Pipeline config, review thresholds
- `{id}-persona.json` - Tone, style, behavioral constraints
- `{id}-kb.md` - Knowledge base (markdown)
- `{id}-kg.jsonld` - Knowledge graph (JSON-LD)

Optional files: `{id}-psi.jsonld` (projections), `engram.jsonld` (session memory)

---

## What Was Built Today

### 1. `engram.py` (NEW FILE - 350+ lines)

Located at: `C:\Users\I820965\dev\aether\engram.py`

**8 Public API Functions:**

| Function | Purpose | Key Details |
|----------|---------|-------------|
| `score_salience(nodes, mentioned_ids, turn_count, kg, decay_rate)` | Compute node salience scores | Direct mention=1.0, Inferred (1-hop neighbor)=0.6, Historical decays by `decay_rate * turns`, Deprecated=0.0 |
| `extract_subgraph(kg, salient_ids, max_hop, max_nodes, salience_scores)` | BFS from salient nodes | Follows any field value matching a known @id as an edge, stops at max_hop, caps at max_nodes |
| `build_manifest(capsule_id, subgraph, salience_scores, turn_id)` | Create JSON-LD manifest | Schema version 1.0, includes @context, active_nodes with salience & hop_depth |
| `save_manifest(manifest, capsule_path)` | Write `engram.jsonld` | Overwrites if exists, returns Path |
| `load_manifest(capsule_path)` | Load engram sidecar | Returns None for cold start |
| `detect_conflict(kg, statement, active_node_ids)` | Lightweight conflict detection | Checks for verbs: sold, no longer, deleted, removed, cancelled, changed, not anymore |
| `warm_context(capsule_path, kg)` | Session START entrypoint | Returns {warm: bool, active_nodes: [], missing_ids: [], manifest: dict\|None} |
| `commit_session(capsule_id, capsule_path, kg, mentioned_ids, turn_id, config)` | Session END entrypoint | Runs full pipeline: score_salience → extract_subgraph → build_manifest → save_manifest |

**Default Config:**
```python
DEFAULT_CONFIG = {
    "max_hop": 2,
    "max_nodes": 20,
    "decay_rate": 0.15,
    "salience_threshold": 0.2,
}
```

**Manifest Schema:**
```json
{
  "@context": {"aether": "...", "engram": "..."},
  "@type": "engram:Manifest",
  "engram:capsule_id": "scholar-buffett",
  "engram:turn_id": "t-042",
  "engram:created": "2026-03-26T00:00:00Z",
  "engram:schema_version": "1.0",
  "engram:source_capsule": "scholar-buffett",
  "engram:active_nodes": [{"@id": "...", "engram:salience": 0.9, "engram:hop_depth": 0}],
  "engram:node_count": 1,
  "engram:max_hop_used": 2
}
```

---

### 2. `kg.py` Modifications

**Added to `add_knowledge()` node creation (lines 106-107):**
```python
"aether:last_accessed": datetime.now().isoformat(),
"aether:access_count": 0,
```

**Added `touch_node()` helper (lines 140-147):**
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

### 3. `stamper.py` Modifications

**Added constant (line 39):**
```python
OPTIONAL_CAPSULE_FILES = ["engram.jsonld"]
```

**Added to `restamp()` function (after line 322):**
```python
# Copy optional sidecar files if present (e.g., engram.jsonld)
for optional_file in OPTIONAL_CAPSULE_FILES:
    optional_path = path / optional_file
    if optional_path.exists():
        shutil.copy(optional_path, new_path / optional_file)
```

---

### 4. `tests/test_engram.py` (NEW FILE)

Located at: `C:\Users\I820965\dev\aether\tests\test_engram.py`

**14 Test Cases (all passing):**
```
TestScoreSalience::test_score_salience_direct_mention      → score == 1.0
TestScoreSalience::test_score_salience_inferred            → score == 0.6
TestScoreSalience::test_score_salience_decay               → score decreases with turn_count
TestScoreSalience::test_score_salience_deprecated_excluded → deprecated nodes always 0.0
TestExtractSubgraph::test_extract_subgraph_max_hop         → nodes beyond max_hop excluded
TestExtractSubgraph::test_extract_subgraph_max_nodes       → result never exceeds max_nodes
TestBuildManifest::test_build_manifest_schema              → all required fields present
TestSaveLoadManifest::test_save_load_manifest_roundtrip    → save then load returns equivalent
TestWarmContext::test_warm_context_cold_start              → warm=False when no engram.jsonld
TestWarmContext::test_warm_context_warm_start              → warm=True, active_nodes populated
TestWarmContext::test_warm_context_missing_ids             → missing_ids populated for stale nodes
TestDetectConflict::test_detect_conflict_verb_pattern      → "I sold the car" conflicts "car" node
TestDetectConflict::test_detect_conflict_no_false_positive → "I bought a car" does not conflict
TestCommitSession::test_commit_session_writes_file         → engram.jsonld created in capsule_path
```

---

## How to Run Tests

```bash
# Self-test (14 inline tests)
python engram.py

# Pytest suite
python -m pytest tests/test_engram.py -v

# All tests
python -m pytest tests/ -v
```

---

## Key Architecture Decisions

1. **No new dependencies** - Standard library + datetime only (as specified)
2. **Imports `kg` module** - `import kg as kg_module` for `get_nodes()`, `mark_deprecated()`, etc.
3. **BFS traversal** - Uses `collections.deque` for efficient breadth-first subgraph extraction
4. **Lightweight conflict detection** - v1 uses verb pattern matching, NLI upgrade deferred
5. **Session-scoped salience** - v1 resets salience each session, multi-session decay deferred

---

## Integration Points

| engram.py calls | Purpose |
|-----------------|---------|
| `kg.load_kg()` | Load KG to traverse |
| `kg.get_nodes()` | Enumerate all nodes |
| `kg.mark_deprecated()` | Tombstone contradicted node (caller responsibility) |
| `kg.mark_updated()` | Update node on non-contradicting refresh |
| `kg.touch_node()` | Update access metadata |

---

## Files NOT Modified (per spec)

- `aec.py` - Triple validation
- `education.py` - Self-education queue
- `aether.py` - Capsule loading

---

## What Is NOT in v1 (Deferred)

| Feature | Reason |
|---------|--------|
| A2A receiver logic | Needs A2A protocol; schema is ready |
| NLI-based conflict detection | v1 uses lightweight verb pattern matching |
| Multi-session decay accumulation | v1 resets salience each session |
| Engram merge (two capsules sharing) | Future shared memory model |
| Engram diff/changelog | Useful, deferred |

---

## Git Status

### aether (code repo)
**Latest Commit:** `8dc1616`
**Branch:** main
**Recent Commits:**
- `8dc1616` - Remove engram session docs (moved to aether-egma repo)
- `ca53171` - Add test suite documentation
- `3804234` - Add deferred-topics.md
- `471ef39` - Add diary entry
- `1915c4f` - Add engram.py - Subgraph Persistence / Working Memory layer

### aether-egma (docs repo)
**Latest Commit:** `a6451e5`
**Branch:** main
**Contents:** Specs + documentation (initialized from local folder)

### Test Suite Status
**Last Run:** 2026-03-27 10:45
**Result:** 49 passed, 12 warnings, 0 failed (4.51s)

| Test File | Tests | Status |
|-----------|-------|--------|
| `test_aec_standalone.py` | 10 | ✓ |
| `test_dai_pulse.py` | 13 | ✓ |
| `test_engram.py` | 14 | ✓ |
| `test_exhaustive.py` | 12 | ✓ |

---

## Spec Reference

The implementation follows `ENGRAM_FEATURE_BRICK_SPEC.md` located in `C:\Users\I820965\dev\aether-egma\`. Key sections:
- Section 3: KG Schema Additions
- Section 5: Public API (8 functions)
- Section 7: Stamper Integration
- Section 9: Test Coverage (13 required cases)
- Section 10: Claude Code Prompt (build instructions)

---

## Next Steps for Future Sessions

1. **Integration with `aether.py`** - Call `warm_context()` on capsule load, `commit_session()` on session end
2. **A2A v2** - Use `engram:source_capsule` for cross-capsule memory passing
3. **NLI conflict detection** - Upgrade from verb patterns to proper NLI model
4. **Multi-session decay** - Track salience across sessions, not just within
5. **Engram visualization** - Add to UI for showing active subgraph

---

## Quick Start for New LLM

```bash
# Two repos to know:
# 1. aether-egma (C:\Users\I820965\dev\aether-egma) - docs, specs, diary
# 2. aether (C:\Users\I820965\dev\aether) - all code

# Navigate to code project
cd C:\Users\I820965\dev\aether

# Verify tests pass
python engram.py                        # 14 inline self-tests
python -m pytest tests/ -v              # Full suite (49 tests)

# Key files to read:
# CODE (in aether/):
#   - engram.py - The new module (8 public API functions)
#   - kg.py - Modified for touch_node() and new fields
#   - stamper.py - Modified for OPTIONAL_CAPSULE_FILES
#   - tests/test_engram.py - 14 test cases
#
# DOCS (in aether-egma/):
#   - ENGRAM_FEATURE_BRICK_SPEC.md - The spec
#   - AETHER_ENGRAM_PROJECT_OVERVIEW.md - Project overview
#   - deferred-topics.md - Future work tracker
#   - aether-engram-test-suite-001.md - Test documentation

# Usage example
import engram
import kg

kg_data = kg.load_kg("path/to/kg.jsonld")
result = engram.warm_context("path/to/capsule", kg_data)
if result["warm"]:
    print(f"Resumed with {len(result['active_nodes'])} active nodes")
else:
    print("Cold start - no prior session")

# At session end
engram.commit_session("capsule-id", "path/to/capsule", kg_data, ["node:1", "node:2"])
```

---

## Session Log

| Time | Action |
|------|--------|
| 23:07 | Started session, read spec |
| 23:15 | Implemented engram.py (8 functions) |
| 23:20 | Modified kg.py (touch_node + new fields) |
| 23:22 | Modified stamper.py (OPTIONAL_CAPSULE_FILES) |
| 23:25 | Created tests/test_engram.py (14 tests) |
| 23:30 | All self-tests passing |
| 23:35 | Pushed to both repos |
| 23:40 | Created diary, deferred-topics, test-suite docs |
| 10:40 | Reorganized repos (code in aether, docs in aether-egma) |
| 10:45 | Final test run: 49 passed |

---

*End of diary entry — Updated 2026-03-27 10:45*
