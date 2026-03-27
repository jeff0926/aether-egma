# AETHER Deferred Topics

Master tracker for all ideas, progress, and status.

**Updated:** 2026-03-26 (rev 16)
**Phase:** 3 active
**Tests:** 355/358 passing (99.2%)

---

## Status Legend

| Symbol | Meaning |
|--------|---------|
| ✅ | Complete |
| 🚧 | In progress |
| 🔒 | Blocked |
| — | Backlog |

---

## Quick Status

### Recently Completed (Session 2026-03-12)
- ✅ Item 38: AEC retry loop + GHOST state
- ✅ Item 39: Two-pass progressive augment
- ✅ Item 40: Capsule export (4 formats)
- ✅ Item 41: CLAUDE.md bidirectional seed
- ✅ Item 42: `aether refine` session sweep

### Blocked Items
- 🔒 **Ψ Layer** (Items 21-25): Needs Kimi session retrieval
- 🔒 **Letter to Anthropic** (Item 54): Needs draft location + personal details

### Known Test Failures (3)
1. Percentage extraction regex ("19.8%" ordering)
2. AEC gate matching (name vs numeric mismatch)
3. StringIO reconfigure (Windows-specific)

---

## AEC Refinements

| # | Item | Status |
|---|------|--------|
| 1 | LLM concept-level verification (Layer 2) | ✅ |
| 2 | Education queue (full cycle) | ✅ |
| 3 | Name matching precision | — |
| 4 | Cross-statement entity resolution | — |
| 5 | Persona ratio monitoring | — |

### AEC Concept Layers (Phase 3a)

| # | Item | Status |
|---|------|--------|
| 5.1 | Layer 1: Compiled pattern matching | ✅ |
| 5.2 | Layer 2: Type-driven LLM verification | ✅ |
| 5.3 | Layer 3: Compiled edge policy traversal | ✅ |
| 5.4 | Anti-gaming fix (strip KG IDs) | ✅ |
| 5.5 | Contradiction gate | ✅ |
| 5.6 | Magnitude extraction ($25 million → 25000000) | ✅ |
| 5.7 | Percentage extraction fix | — |
| 5.8 | Negation detection | — |
| 5.9 | Multi-hop reasoning verification (Layer 4) | — |
| 5.10 | Knowledge decay (pheromone model) | — |
| 5.11 | Education loop poisoning defense | — |

---

## Code Refinements

| # | Item | Status |
|---|------|--------|
| 6 | Refactor augment to use kg.py | — |
| 7 | Statement splitter improvement | — |

---

## KG & Knowledge Management

| # | Item | Status |
|---|------|--------|
| 8 | Selective subgraph extraction (N-hop) | — |
| 9 | Knowledge Origin Classification (5 types) | ✅ |
| 10 | KG maintenance sweeps | — |
| 11 | `aether:source` provenance field | — |

---

## Ingest Pipeline

| # | Item | Status |
|---|------|--------|
| 12 | ingest_skill (SKILL.md → capsule) | ✅ |
| 13 | Three extraction skills | ✅ |
| 14 | Recursive directory ingest | — |
| 15 | Export function (4 formats) | ✅ |
| 16 | Capsule Handshake Protocol | — |

---

## Orchestration & A2A

| # | Item | Status |
|---|------|--------|
| 17 | Orchestrator capsule | ✅ |
| 18 | A2A Subgraph Exchange | — |
| 19 | Auditor capsule | — |
| 20 | Epistemic Rebase | — |

---

## UI / Ψ Layer (🔒 BLOCKED)

| # | Item | Status | Notes |
|---|------|--------|-------|
| 21 | EDS Slivers | 🔒 | Needs Kimi session |
| 22 | CSS Stream | 🔒 | Needs Kimi session |
| 23 | DAI Pulse | 🔒 | Needs Kimi session |
| 24 | 2KB Edge Orchestrator | 🔒 | Needs Kimi session |
| 25 | SAP UDEx Design System Capsule | 🔒 | Needs Kimi session |

---

## Enterprise & Adapters

| # | Item | Status |
|---|------|--------|
| 26 | SAP CAP adapter | — |
| 27 | MCP server adapter | — |
| 28 | REST API adapter | — |
| 29 | AEC as standalone 6th file | — |
| 30 | AEC as MCP server | — |
| 31 | Copilot SKILL.md export | — |
| 32 | CI/CD gate adapter | — |

---

## Dashboard & Reporting

| # | Item | Status |
|---|------|--------|
| 33 | Dashboard (5-tab, port 8864) | ✅ |
| 34 | Dashboard update (orchestrator view) | — |
| 35 | Orchestrator routing visualization | — |
| 36 | AEC Layer visualization | — |

---

## Pipeline Improvements (March 12)

| # | Item | Status |
|---|------|--------|
| 37 | AEC retry loop | ✅ |
| 38 | GHOST state | ✅ |
| 39 | Two-pass progressive augment | ✅ |
| 40 | Capsule export (4 formats) | ✅ |
| 41 | CLAUDE.md bidirectional seed | ✅ |
| 42 | `aether refine` command | ✅ |

---

## Agent Diversity

| # | Item | Status |
|---|------|--------|
| 43 | Scholar agents (Jefferson, Buffett) | ✅ |
| 44 | Skill agents (6 Anthropic skills) | ✅ |
| 45 | Executive roles (8 C-suite) | ✅ |
| 46 | Process agents | — |
| 47 | Reactive agents | — |
| 48 | Grading/calculus agents | — |

---

## Meta / System

| # | Item | Status |
|---|------|--------|
| 49 | CLAUDE.md bidirectional | ✅ |
| 50 | Session versioning | — |
| 51 | Feature brick registry | — |
| 52 | Blind testing | — |
| 53 | Pluggable queue backends | — |

---

## Documentation & Publication

| # | Item | Status | Notes |
|---|------|--------|-------|
| 54 | Letter to Anthropic | 🔒 | Needs draft + personal details |
| 55 | Conceptual piece ("Agents as Skills") | — | |
| 56 | Technical paper | — | |
| 57 | Practitioner piece | — | |

---

## Company Stack (864 Zeros)

| # | Item | Status |
|---|------|--------|
| 58 | Autonomous Application Framework | — |
| 59 | Vulture Nest | — |

---

## Engram Module (engram.py) — Added 2026-03-26

### 1. A2A Receiver Logic
- **Status:** `DEFERRED`
- **Source:** ENGRAM_FEATURE_BRICK_SPEC.md Section 8
- **Description:** Load and process engram manifests from other capsules for cross-agent memory sharing
- **Rationale:** Needs A2A protocol finalization; schema is already A2A-ready with `engram:source_capsule`
- **Dependencies:** A2A v2 protocol spec
- **Implementation Notes:**
  - `warm_context()` could accept optional `source_capsule_path` parameter
  - Merge foreign engram nodes with local KG, mark provenance
  - Handle node ID collisions between capsules
- **Effort Estimate:** Medium
- **Priority:** High (enables agent collaboration)

---

### 2. NLI-Based Conflict Detection
- **Status:** `DEFERRED`
- **Source:** ENGRAM_FEATURE_BRICK_SPEC.md Section 8
- **Description:** Replace lightweight verb pattern matching with proper Natural Language Inference
- **Rationale:** v1 uses simple verb patterns (`sold`, `no longer`, `deleted`, etc.) which has false negatives
- **Current Implementation:** `detect_conflict()` in engram.py lines 200-240
- **Dependencies:** NLI model selection (local vs API)
- **Implementation Notes:**
  - Consider sentence-transformers for local inference
  - Or Claude API call for high-stakes conflicts
  - Cache conflict checks to reduce API costs
  - Threshold tuning needed for precision/recall balance
- **Effort Estimate:** Medium-High
- **Priority:** Medium

---

### 3. Multi-Session Decay Accumulation
- **Status:** `DEFERRED`
- **Source:** ENGRAM_FEATURE_BRICK_SPEC.md Section 8
- **Description:** Track salience decay across sessions, not just within a single session
- **Rationale:** v1 resets salience each session; nodes don't "fade" over multiple sessions
- **Current Implementation:** `score_salience()` uses `turn_count` within session only
- **Dependencies:** None
- **Implementation Notes:**
  - Store `last_session_salience` in engram.jsonld
  - On `warm_context()`, apply cross-session decay based on time delta
  - Add `session_count` or `days_since_access` to decay formula
  - Consider exponential vs linear decay curves
- **Effort Estimate:** Low-Medium
- **Priority:** Medium

---

### 4. Engram Merge (Shared Memory Model)
- **Status:** `DEFERRED`
- **Source:** ENGRAM_FEATURE_BRICK_SPEC.md Section 8
- **Description:** Allow two capsules to share/merge engram manifests for collaborative memory
- **Rationale:** Future shared memory model for agent teams
- **Dependencies:** A2A protocol, conflict resolution strategy
- **Implementation Notes:**
  - Define merge semantics (union, intersection, weighted)
  - Handle salience conflicts (same node, different scores)
  - Provenance tracking for merged nodes
  - Consider "engram channels" for topic-based sharing
- **Effort Estimate:** High
- **Priority:** Low (future capability)

---

### 5. Engram Diff / Changelog
- **Status:** `DEFERRED`
- **Source:** ENGRAM_FEATURE_BRICK_SPEC.md Section 8
- **Description:** Track changes between engram sessions for debugging and analysis
- **Rationale:** Useful for understanding memory evolution, but not critical for v1
- **Dependencies:** None
- **Implementation Notes:**
  - Store previous engram as `engram.prev.jsonld` or embed diff in manifest
  - Track: nodes added, nodes dropped, salience changes
  - Could enable "memory replay" for debugging
- **Effort Estimate:** Low
- **Priority:** Low

---

## Knowledge Graph (kg.py)

### 6. Node Relationship Indexing
- **Status:** `DEFERRED`
- **Source:** Performance observation during engram implementation
- **Description:** Build adjacency index for faster neighbor lookups
- **Rationale:** Current BFS in `extract_subgraph()` scans all nodes for each edge lookup
- **Dependencies:** None
- **Implementation Notes:**
  - Build `{node_id: [neighbor_ids]}` index on KG load
  - Invalidate/rebuild on `add_knowledge()`, `mark_deprecated()`
  - Trade memory for speed on large KGs
- **Effort Estimate:** Low
- **Priority:** Low (optimize when needed)

---

### 7. KG Compaction / Garbage Collection
- **Status:** `DEFERRED`
- **Source:** Identified during engram design
- **Description:** Remove deprecated nodes after configurable retention period
- **Rationale:** Deprecated nodes accumulate; never cleaned up currently
- **Dependencies:** None
- **Implementation Notes:**
  - Add `compact_kg(kg, max_age_days=30)` function
  - Only remove deprecated nodes older than threshold
  - Preserve nodes referenced by non-deprecated nodes
- **Effort Estimate:** Low
- **Priority:** Low

---

## Stamper (stamper.py)

### 8. Engram Export in All Formats
- **Status:** `DEFERRED`
- **Source:** Identified during stamper modification
- **Description:** Include engram summary in `claude-md`, `github-agent-md`, `a2a-agent-card` exports
- **Rationale:** Exports currently don't reflect session memory state
- **Dependencies:** engram.py complete (DONE)
- **Implementation Notes:**
  - Add "Recent Context" or "Working Memory" section to exports
  - Include top-N salient nodes with labels
  - Truncate for token limits in skill exports
- **Effort Estimate:** Low
- **Priority:** Medium

---

## UI / Visualization

### 9. Engram Subgraph Visualization
- **Status:** `DEFERRED`
- **Source:** Diary entry 2026-03-26
- **Description:** Visual display of active subgraph in UI
- **Rationale:** Helps users understand what the agent is "thinking about"
- **Dependencies:** UI framework decisions
- **Implementation Notes:**
  - Graph visualization (D3.js, vis.js, or similar)
  - Node size = salience, color = hop depth
  - Click node to see full KG entry
  - Animate salience decay in real-time
- **Effort Estimate:** Medium-High
- **Priority:** Medium (UX improvement)

---

## Ideas Backlog

| Idea | Source | Notes |
|------|--------|-------|
| Salience-weighted RAG retrieval | 2026-03-26 | Use engram salience to boost/demote RAG results |
| Engram compression | — | Delta-encode engrams to reduce storage |
| Cross-capsule node linking | — | Reference nodes from other capsules by URI |
| Temporal engram snapshots | — | Keep N historical engrams for "memory replay" |
| Forgetting curve customization | — | Per-capsule decay curves |
| SHACL distinction article | — | AEC validates NL, SHACL validates structured data |
| Cross-LLM validation | — | Different LLMs for generation vs verification |

---

## CLI Surface (Current)

```bash
aether stamp <name> [--source <file>] [--output <dir>] [--version <ver>]
aether run <capsule> <query> [--provider <p>] [--model <m>]
aether validate <capsule>
aether info <capsule>
aether queue <capsule>
aether educate <capsule> <record_id> [--provider <p>]
aether verify <text> <capsule>
aether export <capsule> --format <format> [--output <path>]
aether refine <capsule> [--n N] [--auto-queue]
```

---

## Completed Items (Recent)

| Item | Completed | Notes |
|------|-----------|-------|
| Engram v1 Implementation | 2026-03-26 | engram.py with 8 public API functions |
| KG touch_node() helper | 2026-03-26 | Added to kg.py |
| Stamper OPTIONAL_CAPSULE_FILES | 2026-03-26 | engram.jsonld included in restamp |
| CLAUDE.md bidirectional | 2026-03-12 | Import/export working |
| Capsule export (4 formats) | 2026-03-12 | skill-md, claude-md, json, jsonld |
| `aether refine` command | 2026-03-12 | Session analysis for KG candidates |
| GHOST state | 2026-03-12 | Unverifiable responses marked 0.0 |
| Two-pass augment | 2026-03-12 | Headers first, then content |
| AEC retry loop | 2026-03-12 | Regenerate on first fail |

---

*Rev 16 — 2026-03-26*
*Previous: Rev 15 (2026-03-20)*
