# AETHER Engram — Project Overview & Architecture
**Module:** `engram.py`  
**Project:** AETHER (Adaptive Embodied Thinking Holistic Evolutionary Runtime)  
**Owner:** 864zeros LLC  
**Version:** 1.0  
**Status:** Spec complete. Build ready.  
**Date:** 2026-03-26

---

## What This Document Is

This document is the authoritative reference for understanding `engram.py` — what it is, why it exists, how it works, and how it fits into the AETHER system. It is written for Jeff Conn (principal architect) and for any LLM that needs to reason about this module without prior session context.

It is NOT a README. It is an architecture and context document. The build spec is in `ENGRAM_FEATURE_BRICK_SPEC.md`.

---

## 1. The Problem It Solves

### The Cold Start Problem

Every time an AETHER capsule processes a query, it starts from zero. The pipeline runs — distill, augment, generate, review — and produces a response. Then the context disappears. The next query arrives cold. The agent has no memory of what just happened.

This is fine for single-turn use. It breaks down the moment you need:
- Multi-turn conversations where earlier context matters
- Agents that hand off work to other agents
- Sessions that resume after interruption
- Any scenario where "what was this agent thinking about?" is a meaningful question

### The Token Blob Problem

The naive fix is to prepend a conversation summary to every prompt. This works at small scale and collapses at large scale. Summaries are:
- Lossy — detail is dropped to fit token budgets
- Unverifiable — there is no way to AEC-check a free-text summary
- Expensive — the token cost grows with every turn
- Disconnected — summaries are not grounded in the KG; they float free

### The Engram Solution

Instead of passing a text summary, pass a **list of KG node IDs** — the nodes that were active, weighted by how relevant they were. The next turn loads those nodes from the KG directly. The context is:
- Lossless at the knowledge level — node URIs point to full structured data
- Verifiable — nodes are KG-grounded, AEC-compatible
- Efficient — a manifest of 20 node IDs is ~1KB vs. potentially thousands of tokens of summary text
- Portable — the manifest is JSON-LD and travels with the capsule

This is the neuroscience parallel: the brain doesn't store a summary of every experience. It stores a **trace** — a pointer to the neural pattern — and reconstructs from there. An engram is that trace. Hence the name.

---

## 2. Where Engram Sits in AETHER

### The AETHER Stack (Quick Reference)

```
AETHER Core Stack
─────────────────────────────────────────────────────────
aether.py       Capsule class. Loads 5 files. Runs 4-stage pipeline.
                distill → augment → generate → review

aec.py          Agent Education Calibration.
                Verifies responses are grounded in KG. Scores 0.0–1.0.

education.py    Self-education loop.
                AEC failure → research → integrate → re-evaluate.

kg.py           Knowledge Graph.
                JSON-LD. 5 origin types: core/acquired/updated/deprecated/provenance.

llm.py          LLM abstraction. Anthropic + OpenAI + stub.

habitat.py      Capsule registry. Multi-capsule routing.

stamper.py      Capsule factory. Creates 5-file folders with lineage.

report.py       ASCII execution report.

cli.py          Command-line interface.
─────────────────────────────────────────────────────────
engram.py       ← THIS MODULE
                Subgraph Persistence / Working Memory layer.
                Sits alongside the pipeline. Activates at session boundaries.
─────────────────────────────────────────────────────────
```

### Where Engram Activates in the Pipeline

Engram is **not** a pipeline stage. It is a session-boundary layer that wraps the pipeline:

```
SESSION START
│
├── engram.warm_context()
│   └── Load engram.jsonld if it exists
│       └── Hydrate active nodes from KG
│       └── Inject into augment stage context
│
│   [4-stage pipeline runs normally]
│   distill → augment → generate → review
│                ↑
│       engram active nodes available here
│       (passed to augment as pre-warmed KG context)
│
SESSION END
│
└── engram.commit_session()
    └── Score node salience for this session
    └── Extract active subgraph (max-hop BFS)
    └── Build Engram Manifest (JSON-LD)
    └── Write engram.jsonld to capsule folder
```

Engram does not replace the augment stage. It **pre-populates** it. The augment stage still runs its KB paragraph matching and KG entity query — engram just ensures that nodes that mattered in prior turns are already present in the context before augment begins.

---

## 3. The Capsule and the Sidecar

### The 5-File Capsule (Unchanged)

AETHER capsules are folders. Each folder contains exactly 5 files:

```
scholar-buffett/
├── scholar-buffett-manifest.json     Agent identity + version
├── scholar-buffett-definition.json   Pipeline config + behavior rules
├── scholar-buffett-persona.json      Tone, style, voice
├── scholar-buffett-kb.md             Prose knowledge base
└── scholar-buffett-kg.jsonld         Knowledge graph (JSON-LD)
```

This 5-file contract is **settled architecture**. Engram does not change it.

### The Optional Sidecar

Engram adds a **6th optional file** that is generated, not authored:

```
scholar-buffett/
├── scholar-buffett-manifest.json
├── scholar-buffett-definition.json
├── scholar-buffett-persona.json
├── scholar-buffett-kb.md
├── scholar-buffett-kg.jsonld
└── engram.jsonld                     ← Generated at session end. Optional.
```

`engram.jsonld` is:
- **Present** when a session has been committed
- **Absent** on first run (cold start — agent starts fresh)
- **Overwritten** at the end of each session (last session wins)
- **Included** in stamper zip archives when present
- **Ignored** by all pipeline logic if absent — no breaking change

The precedent for this pattern is `psi.jsonld` — the UI projection sidecar settled in prior architectural review. Optional sidecars are an established AETHER pattern.

---

## 4. The Engram Manifest Format

The manifest is JSON-LD. It is designed to be A2A-ready from day one — when agent-to-agent handoff is implemented (future), no schema changes are needed.

```json
{
  "@context": {
    "aether": "http://aether.dev/ontology#",
    "engram": "http://aether.dev/engram#"
  },
  "@type": "engram:Manifest",
  "engram:capsule_id": "scholar-buffett",
  "engram:source_capsule": "scholar-buffett",
  "engram:turn_id": "t-042",
  "engram:created": "2026-03-26T12:00:00Z",
  "engram:schema_version": "1.0",
  "engram:active_nodes": [
    {
      "@id": "aether:core/warren_buffett",
      "engram:salience": 0.9,
      "engram:hop_depth": 0
    },
    {
      "@id": "aether:core/berkshire_hathaway",
      "engram:salience": 0.6,
      "engram:hop_depth": 1
    }
  ],
  "engram:node_count": 2,
  "engram:max_hop_used": 2
}
```

### Key Fields

| Field | Purpose |
|-------|---------|
| `engram:source_capsule` | Which capsule generated this manifest — critical for A2A |
| `engram:active_nodes` | The persisted subgraph — node IDs + salience scores + hop depth |
| `engram:salience` | How "hot" this node was — 0.0 (cold) to 1.0 (directly mentioned) |
| `engram:hop_depth` | How many edges from a directly-mentioned node — 0 = direct, 1 = neighbor, 2 = neighbor's neighbor |
| `engram:schema_version` | Forward compatibility — receiver can handle schema evolution |

---

## 5. Node Salience — How Memory Fades

Salience is the mechanism by which engram decides what to remember and what to let fade. It mirrors trace decay in neuroscience.

### Salience Tiers

| Tier | Score | Condition |
|------|-------|-----------|
| Direct mention | `1.0` | Node `@id` explicitly activated this turn |
| Inferred context | `0.6` | Neighbor of a directly-mentioned node (within 1 hop) |
| Historical | Decaying | Node accessed in prior turns, not this one |
| Unaccessed | `0.0` | Never touched — excluded from manifest |
| Deprecated | `0.0` | Always excluded regardless of prior salience |

### Decay Formula

```
salience = max(0.0, prior_score - decay_rate × turns_since_access)
```

Default `decay_rate = 0.15`. A node last accessed 4 turns ago with prior salience `0.9`:

```
0.9 - (0.15 × 4) = 0.9 - 0.6 = 0.3
```

Still above the default `salience_threshold = 0.2` — it survives in the manifest. One more turn without mention:

```
0.3 - 0.15 = 0.15 → below threshold → pruned from manifest
```

This is **trace decay**: memories that stop being activated fade until they drop below the threshold and are no longer included in active context. The KG node itself is never deleted — only the engram stops pointing to it.

---

## 6. Max-Hop Traversal — Spreading Activation

The neuroscience concept of **spreading activation** says that activating one memory node activates related nodes. Engram implements this via BFS traversal.

```
max_hop = 2 (default)

Direct mention: Warren Buffett node (hop 0, salience 1.0)
    │
    ├── Berkshire Hathaway node (hop 1, salience 0.6)
    │       │
    │       └── GEICO node (hop 2, salience 0.4)
    │
    └── Value Investing node (hop 1, salience 0.6)
            │
            └── Benjamin Graham node (hop 2, salience 0.4)
```

Anything at hop 3 is excluded. This prevents the entire KG from flooding into context for any well-connected node.

`max_nodes = 20` is a hard cap. If BFS returns 35 nodes, keep the 20 with highest salience. Token efficiency is the constraint.

Both `max_hop` and `max_nodes` are configurable per capsule in `definition.json` under the `"engram"` block.

---

## 7. Conflict Detection — Memory Reconsolidation

When a user says something that contradicts a persisted node, engram detects it and triggers the tombstone process.

### The Pattern

```
Persisted node:  aether:core/car → "user owns a 2021 Honda Civic"
New statement:   "I sold the car last month"

Conflict detected → mark_deprecated(node_id, reason="engram_conflict")
```

The deprecated node remains in the KG (AETHER never deletes — only deprecates). It is immediately scored `0.0` and excluded from all future manifests and augment context.

### v1 Detection Method

Version 1 uses lightweight verb pattern matching against active node labels:
- Trigger verbs: `sold`, `no longer`, `deleted`, `removed`, `cancelled`, `changed`, `not anymore`
- If trigger verb appears AND an active node label appears in the same statement → conflict

This is intentionally simple. NLI-based conflict detection (FOLLOW/VIOLATE classification) is deferred to a future version — the function signature and integration point are already designed to accept it as a drop-in upgrade.

---

## 8. KG Schema Additions

Two new fields are added to KG nodes. These are **additive only** — existing nodes without them are handled gracefully.

```json
{
  "@id": "aether:core/warren_buffett",
  "rdfs:label": "Warren Buffett",
  "aether:origin": "core",
  "aether:confidence": 1.0,
  "aether:acquired_date": "2026-03-01T00:00:00Z",
  "aether:last_accessed": "2026-03-26T12:00:00Z",
  "aether:access_count": 7
}
```

| New Field | Type | Purpose |
|-----------|------|---------|
| `aether:last_accessed` | ISO datetime | When this node was last activated — powers decay calculation |
| `aether:access_count` | Integer | How many times this node has been activated — future reinforcement signal |

A new helper `touch_node(kg, node_id)` in `kg.py` increments these when a node is activated.

---

## 9. The `definition.json` Engram Config Block

Capsule authors can tune engram behavior per-capsule. If the block is absent, defaults apply.

```json
{
  "pipeline": { ... },
  "engram": {
    "max_hop": 2,
    "max_nodes": 20,
    "decay_rate": 0.15,
    "salience_threshold": 0.2
  }
}
```

A capsule that handles dense, interconnected knowledge (e.g., a legal reference agent) might use `max_hop: 1` and `max_nodes: 10` to stay tight. A research agent might use `max_hop: 3` and `max_nodes: 40`.

---

## 10. Portability and A2A Readiness

### Within a Single Capsule

The engram sidecar makes sessions resumable. A capsule that was mid-conversation yesterday picks up where it left off today. The manifest tells it which nodes to warm before the first augment stage runs.

### Across Capsules (A2A — Future v2)

The `engram:source_capsule` field means the manifest is already portable. When Agent A hands off to Agent B:

1. Agent A commits its session → writes `engram.jsonld`
2. Handoff payload includes `engram.jsonld`
3. Agent B calls `warm_context()` with Agent A's manifest
4. Agent B's `warm_context()` cross-references its own KG — nodes that match load hot, nodes that don't exist in Agent B's KG are logged as `missing_ids`
5. Agent B starts warm on shared knowledge, cold on Agent A's private knowledge

This is **transferable context**, not transferable memory. Agent B doesn't absorb Agent A's KG. It only activates the nodes it already has that A flagged as relevant. The result: Agent B starts the conversation knowing what A considered important, without inheriting A's identity.

No schema changes are needed for this. The v1 implementation already supports it — only the receiver-side loading logic needs to be wired in v2.

---

## 11. What Engram Does NOT Do

Important boundaries — these prevent scope creep:

| Out of Scope | Why |
|---|---|
| Storing conversation history | That's a transcript, not a memory trace |
| Replacing the augment stage | Engram pre-warms augment; it doesn't replace KB/KG lookup |
| Modifying AEC scoring | AEC is the validator; engram is the context layer |
| Cross-session decay accumulation | v1 resets each session; multi-session decay is future |
| Managing the education queue | education.py owns that; engram only triggers conflict detection |
| A2A receiver logic | v2 feature; schema is ready, wiring is not |

---

## 12. Intersection With Other AETHER Modules

| Module | Relationship |
|--------|-------------|
| `kg.py` | Engram reads from and writes to the KG. Calls `load_kg`, `get_nodes`, `mark_deprecated`, `mark_updated`, `touch_node` (new). |
| `aether.py` | Engram wraps the pipeline at session boundaries. The `Capsule.run()` method does not call engram internally — it is called by the session manager or CLI around `run()`. |
| `education.py` | No direct call. But both share the same conflict signal: education detects acquired node contradictions against core nodes during self-education; engram detects conversational contradictions against active nodes in real-time. These are parallel, not overlapping. |
| `aec.py` | No direct call. AEC verifies generated responses. Engram manages context. They operate on different moments in the pipeline. |
| `stamper.py` | Stamper includes `engram.jsonld` in archives when present. One-line addition to `OPTIONAL_CAPSULE_FILES`. |
| `habitat.py` | No interaction in v1. In v2, habitat may load engram manifests when routing between capsules. |

---

## 13. File Summary

| File | Status | Notes |
|------|--------|-------|
| `engram.py` | **TO BUILD** | New module. 8 public functions. Standard library + datetime only. |
| `kg.py` | **TO MODIFY** | Add `touch_node()` helper + 2 new node fields in `add_knowledge()`. |
| `stamper.py` | **TO MODIFY** | Add `engram.jsonld` to `OPTIONAL_CAPSULE_FILES` list. |
| `engram.jsonld` | **GENERATED** | Optional sidecar. Written by `commit_session()`. |
| `tests/test_engram.py` | **TO BUILD** | 13 test cases. See build spec Section 9. |

---

## 14. Decision Log

| Decision | Rationale |
|----------|-----------|
| Optional sidecar, not 6th core file | Preserves 5-file contract. Follows `psi.jsonld` precedent. Engram is session state, not agent identity. |
| JSON-LD manifest format | Already the KG format. Portable. A2A-compatible without schema changes. |
| v1: Session persistence only | A2A is a bigger feature. Do one thing well first. Schema is A2A-ready regardless. |
| v1: Verb pattern conflict detection | NLI is the right long-term answer. Verb matching is fast, testable, and honest about its limits. |
| `max_hop = 2` default | 1 is too narrow. 3 risks pulling 80%+ of a well-connected graph. 2 is the sweet spot. |
| Named "Engram" | A neuroscience engram is a physical memory trace — exactly what this system manages. Neutral/functional tone. Earned, not decorative. |
| Standard library only | No new dependencies. Follows AETHER's zero-dependency-creep policy. |

---

## 15. Glossary

| Term | Definition |
|------|-----------|
| **Engram** | A physical memory trace — the neural pattern that persists after a memory is formed. In AETHER: the persisted active subgraph from a session. |
| **Engram Manifest** | The JSON-LD sidecar file (`engram.jsonld`) containing the active subgraph, salience scores, and session metadata. |
| **Salience** | How "hot" a node is — a 0.0–1.0 score representing how relevant it was to the current session. |
| **Trace Decay** | The mechanism by which salience decreases for nodes not activated in recent turns. |
| **Spreading Activation** | The BFS traversal from directly-mentioned nodes — nearby nodes inherit partial salience. |
| **Max-Hop** | The maximum number of edges the BFS traversal follows from a salient node. |
| **Warm Start** | Session begins with a prior engram manifest loaded — context is pre-populated. |
| **Cold Start** | No prior manifest exists — session begins fresh from the KG. |
| **Tombstone** | Marking a KG node as `deprecated` — it remains in the KG but is scored 0.0 and excluded from all context. AETHER never deletes. |
| **A2A** | Agent-to-Agent — future capability where one capsule passes its engram manifest to another during task handoff. |
| **touch_node** | New `kg.py` helper that increments `aether:access_count` and updates `aether:last_accessed` when a node is activated. |

---

*Document authored by Tom (Temporal Orchestration Mechanism)*  
*AETHER Session 2026-03-26 — 864zeros LLC*
