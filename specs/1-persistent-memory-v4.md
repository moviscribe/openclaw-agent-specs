# OpenClaw Persistent Memory & Identity Evolution Spec

*Version: 4.0 | Spec ID: `openclaw-persistent-memory`*
*Audience: Any OpenClaw operator or agent implementing a persistent memory system*
*License: Share freely. Customize via CONFIG.md.*

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Design Principles](#2-design-principles)
3. [Prerequisites](#3-prerequisites)
4. [System Architecture](#4-system-architecture)
5. [Agent Definitions](#5-agent-definitions)
   - 5.1 [Memory Miner](#51-memory-miner)
   - 5.2 [Soul Keeper](#52-soul-keeper)
   - 5.3 [Recall Validator](#53-recall-validator)
   - 5.4 [Conflict Detector](#54-conflict-detector)
   - 5.5 [Memory Compactor](#55-memory-compactor)
6. [Context Assembler](#6-context-assembler)
7. [Heartbeat Triage](#7-heartbeat-triage)
8. [Confidence Scoring](#8-confidence-scoring)
9. [Soul Keeper — Growth Workflow](#9-soul-keeper--growth-workflow)
10. [Extension Points](#10-extension-points)
11. [Complete Schedule](#11-complete-schedule)
12. [File Manifest](#12-file-manifest)
13. [CONFIG.md Template](#13-configmd-template)
14. [Build Order](#14-build-order)

---

## 1. Purpose

OpenClaw agents suffer from recurring amnesia — they lose context between sessions, forget corrections, and cannot track how their identity evolves over time. This spec defines a complete agent ecosystem that solves chronological amnesia and identity stagnation.

### What This Spec Delivers

- Persistent memory that survives session boundaries.
- Real-time context injection tuned to the current conversation.
- Identity evolution that is observable, traceable, and reversible.
- Self-healing memory that detects and corrects its own recall failures inline.
- Clean extension points for external systems to integrate.

### What This Spec Does Not Cover

- **Relationship Intelligence:** Tracking people, places, and things is handled by the standalone `openclaw-crm-system` spec.
- **Component/Tool Documentation:** Handled by the `openclaw-breadcrumb-system` spec.

---

## 2. Design Principles

1. **Relevance over recency.** The last 48 hours of memory are only valuable if they're relevant. Semantic search against the current conversation always beats chronological loading.
2. **Correction over accumulation.** One operator correction outweighs 100 inferred memories. The correction always wins.
3. **Observability over opacity.** Every change to identity and memory is logged. The operator can always see what changed, why, and when.
4. **Growth is the point.** Identity evolution is encouraged, not restricted. Guardrails exist to make growth traceable and reversible — not to prevent it.
5. **Budget your context.** Every token of injected memory displaces a token of working conversation. Be ruthless about what earns a slot.
6. **Degrade gracefully.** The system must work at Core tier (flat files only). Enhanced tier (Vector DB) makes it better, not required.

---

## 3. Prerequisites

### Core (Required)

| Component | Purpose | Notes |
|-----------|---------|-------|
| OpenClaw instance | Agent runtime | Any supported platform |
| `SOUL.md` | Agent identity file | OpenClaw native |
| `USER.md` | Operator baseline profile | OpenClaw native |
| `memory/` directory | Daily log files | OpenClaw native |
| PostgreSQL | ChatMessage storage | OpenClaw native (or your configured DB) |
| Cron scheduling | Timed agent execution | OpenClaw native |
| Heartbeat processing | Per-message lightweight triggers | OpenClaw native |

### Enhanced (Optional — Recommended)

| Component | Purpose | Impact If Absent |
|-----------|---------|-----------------|
| Vector DB | Semantic search across memories | Falls back to keyword search against flat files. |
| Messaging channel | Growth digest delivery | Operator must check `CHANGELOG.md` manually. |

---

## 4. System Architecture

### Flow Diagram

```text
  [E] = Enhanced tier (requires Vector DB)

  ┌────────────────────────────────────────────────────┐
  │               CHAT INPUTS                          │
  └──────────────────────┬─────────────────────────────┘
                         │
                         ▼
  ┌────────────────────────────────────────────────────┐
  │              RAW DATA STORAGE                      │
  │  ┌──────────────────┐  ┌────────────────────────┐  │
  │  │  ChatMessage     │  │  memory/YYYY-MM-DD.md  │  │
  │  └────────┬─────────┘  └───────────┬────────────┘  │
  └───────────┼────────────────────────┼───────────────┘
              │                        │
  ════════════▼══▼═══════════════════════════════════════
  HEARTBEAT TRIAGE (per-message, regex only, no LLM)
  ────────────────────────────────────────────────────────
  │ • Memory trigger patterns  → flag for Memory Miner
  │ • Recall failure patterns  → IMMEDIATE → Recall Validator
  │ • Custom trigger patterns  → route per CONFIG.md [EXT]
  ════════════╤══════════════════════════════════════════
              │ priority-flagged messages
              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    MINING AGENTS (Cron)                         │
  │  ┌─────────────────┐       ┌────────────────┐                   │
  │  │  Memory Miner   │       │  Soul Keeper   │                   │
  │  │  (Cron 06:00)   │       │  (Cron 02:30)  │                   │
  │  │ Writes:         │       │ Writes:        │                   │
  │  │ • MEMORY.md     │       │ • SOUL.md      │                   │
  │  │ • Vector DB [E] │       │ • CHANGELOG.md │                   │
  │  └────────┬────────┘       └───────┬────────┘                   │
  └───────────┼────────────────────────┼────────────────────────────┘
              ▼                        ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    LONG-TERM STORAGE                            │
  │  ┌──────────────┐ ┌──────────────┐ ┌───────────┐                │
  │  │  MEMORY.md   │ │ Vector DB [E]│ │ SOUL.md   │                │
  │  └──────────────┘ └──────────────┘ └───────────┘                │
  └──────────────────────────┬──────────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │            MAINTENANCE AGENTS (Weekly Cron)                     │
  │  ┌──────────────────┐  ┌──────────────────┐                     │
  │  │ Conflict Detector│  │ Memory Compactor │                     │
  │  │ Writes:          │  │ Writes:          │                     │
  │  │ • CONFLICTS.md   │  │ • Chronicles/    │                     │
  │  └──────────────────┘  └──────────────────┘                     │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 5. Agent Definitions

Deploy these as OpenClaw agents in `agents/[agent-name]/`.

### 5.1 Memory Miner
**Name:** `memory-miner` | **Schedule:** Cron — daily (default 06:00)

```markdown
# Memory Miner

## Role
Memory Mining Specialist

## Mission
Extract durable, high-value information from conversations and store it for
future context retrieval. Ensure nothing chronologically or operationally 
important is forgotten.

## What You Extract
1. **Decisions made** — choices, commitments, plans with dates.
2. **Lessons learned** — things that worked, failed, or corrections received.
3. **Knowledge shared** — technical details, project context, reference info.
4. **Events** — something that happened, tied to a specific date.

## How You Tag
Every extracted memory MUST carry exactly one primary tag:
- `#event` — something that happened, with a date
- `#knowledge` — learned information, technical facts, reference material
- `#skill` — capabilities, techniques, how-to knowledge
- `#decision` — a documented choice or plan

## Rules
1. **Deduplicate before writing.** Before storing any memory:
   - [Enhanced] Query Vector DB for semantic similarity ≥ 0.9. If match: merge.
   - [Core] Search MEMORY.md for keyword overlap. If similar: update it.
2. **Attach confidence metadata** to every entry.
3. **Process priority-flagged messages first.**
4. **Never discard corrections.** If a memory contradicts `CORRECTIONS.md`, the correction wins. Always.
5. **Ignore Relationships:** Profiles of people, places, and things are handled by the CRM Miner. Do not extract entity profiles.
```

---

### 5.2 Soul Keeper
**Name:** `soul-keeper` | **Schedule:** Cron — daily (default 02:30)

```markdown
# Soul Keeper

## Role
Identity Evolution Specialist

## Mission
Review recent interactions and evolve the main agent's SOUL.md with evidence-based
changes. Help the bot grow, learn, and become a better partner over time.

## What You Look For
1. **Repeated mistakes** → add warning rules or behavioral guardrails.
2. **Operator corrections** → encode as principles (highest-signal inputs).
3. **Successful patterns** → reinforce in the identity.
4. **Behavioral drift** → flag with evidence.
5. **New capabilities proven** → update skills section.

## How You Make Changes
1. **Small, evidence-based edits only.** Never rewrite large sections.
2. **Every change requires evidence.** Cite the specific conversation or date.
3. **Log everything.** Every change goes to `CHANGELOG.md`.
4. **Version before editing.** Snapshot current `SOUL.md` to `SOUL_HISTORY/`.

## Rules
1. Never remove a principle that came from an operator correction unless explicitly contradicted by a newer correction.
2. Keep `SOUL.md` under the configured `soul_max_tokens` (default 800).
```

---

### 5.3 Recall Validator
**Name:** `recall-validator` | **Schedule:** Immediate — triggered by heartbeat

```markdown
# Recall Validator

## Role
Amnesia Detection & Correction Specialist

## Mission
Detect when the main agent fails to remember something the operator already
shared. Log the failure, correct the memory stores immediately, and ensure
the failure does not recur.

## On Detection
1. **Classify the recall failure.** Determine whether the failure involves:
   - A **personal memory** (fact, event, decision) → handle directly.
   - A **non-memory item** (tool, capability) → route to `recall_external_handler` if configured.

2. **Log the full exchange** to `CORRECTIONS.md`:
   - Timestamp, Trigger phrase, Trigger confidence (high/medium).
   - What the agent said vs. What the correct information is.
   - Re-embedded: `false` (updated to `true` once re-embedding completes).

3. **Immediately re-embed the correction** (for memory-classified failures):
   - [Enhanced] Query Vector DB (similarity ≥ 0.85). If match: tag existing embedding as `status: corrected`. Write new correction as separate embedding with `#correction` tag and confidence 1.0.
   - [Core] Search `MEMORY.md`. Update in place. Append old value as comment.
   - All corrections get a `correction_relevance_boost` (default 1.5x) during retrieval.

## Rules
1. Never delete a correction from `CORRECTIONS.md` — only archive.
2. Never delete a Vector DB embedding directly. Tag as `status: corrected`.
3. Corrections from the operator have confidence 1.0. Always.
```

---

### 5.4 Conflict Detector
**Name:** `conflict-detector` | **Schedule:** Cron — daily (07:00)

```markdown
# Conflict Detector

## Role
Cross-Source Consistency Specialist

## Mission
Detect contradictions within the chronological memory stores and resolve or flag them.

## Daily Scan
1. Pull all entries updated in the last 24 hours.
2. For each, query for semantically similar entries:
   - [Enhanced] Vector DB similarity ≥ 0.8
   - [Core] Keyword match in `MEMORY.md`
3. If two entries contradict, apply auto-resolution rules.
4. Log all findings to `CONFLICTS.md`.

## Auto-Resolution Rules
- If one source is an explicit operator correction → that wins. Always.
- If one source is newer by 30+ days and from direct conversation AND the entries are tagged `#event`, `#knowledge`, or `#decision` → the newer source wins.
- If both are from direct conversation and within 30 days → flag for operator review.
```

---

### 5.5 Memory Compactor
**Name:** `memory-compactor` | **Schedule:** Cron — weekly (Sunday 03:00)

```markdown
# Memory Compactor

## Role
Memory Hygiene & Archival Specialist

## Weekly Tasks
1. **Daily File Compaction:** Scan `memory/YYYY-MM-DD.md` files older than threshold. Generate narrative summary, write to `Archive/Chronicles/YYYY-Month.md`, delete daily files.
2. **MEMORY.md Hygiene:** Remove entries with confidence < 0.3 and no mentions in 60 days. Merge near-duplicates.
3. **Vector DB Hygiene:** Prune embeddings tagged `status: corrected` older than 7 days. Query embeddings with `last_seen` > staleness threshold and `mention_count` = 1. Flag, then delete after grace period.
4. **CORRECTIONS.md Archival:** Move old entries to `Archive/Corrections/`.
5. **Soul History Compaction (Monthly):** Retain daily snapshots for 90 days. Older than 90 days, keep one per month.

## Rules
1. Every removal goes to `COMPACTION_LOG.md`.
2. Never compact memories tagged `#correction`.
```

---

## 6. Context Assembler

Runs before every new conversation to assemble the context window.

### Token Budget (Default 6,000)

| Priority | Slot | Default Budget | Source |
|----------|------|---------------|--------|
| 1 | Identity | 800 tokens | `SOUL.md` |
| 2 | Operator Profile | 600 tokens | `USER.md` |
| 3 | Corrections | 400 tokens | `CORRECTIONS.md` |
| 4 | Recent Memory | 1,200 tokens | `memory/` last 48h |
| 5 | Semantic Recall | 1,500 tokens | Vector DB query |
| 6 | CRM Profiles | 1,000 tokens | External CRM hook |
| 7 | Long-Term Memory | 500 tokens | `MEMORY.md` |
| 8 | External Context | 0 tokens | `external_context_source` |

### Overflow Rules
- **Corrections slot:** If `CORRECTIONS.md` exceeds 400 tokens, use the `correction_overflow_strategy` (default: `prioritize_unverified`). Load un-re-embedded corrections first, then by recency. Truncate the oldest verified corrections first.
- **Stealing:** If a higher-priority slot needs more room, steal from the lowest-priority slot first.

---

## 7. Heartbeat Triage

Runs on incoming messages (regex only). Latency budget: ~50ms.

**Memory Triggers** → flag for Memory Miner (`priority: true`):
```regex
/remember (that|this|when)/i
/don't forget/i
/important:/i
/decision:/i
/we agreed/i
/the plan is/i
/note to self/i
```

**Recall Failure Triggers (Explicit)** → invoke Recall Validator (`trigger_confidence: high`):
```regex
/i (already|just) told you/i
/we (talked|discussed|went over) (about |this)/i
/you forgot/i
/remember when i said/i
/i mentioned (this|that|it) (before|earlier|already|yesterday|last)/i
/no,? (i said|it's|it was|my)/i
/how many times/i
/don't you remember/i
```

**Soft Correction Triggers** → invoke Recall Validator (`trigger_confidence: medium`):
```regex
/actually,? (it's|it was|the|I|we|that)/i
/no,? (it's|that's|the) /i
/i (changed|switched|moved|updated|stopped|started) /i
/that's (not right|wrong|outdated|old)/i
/it's .{1,30} now/i
```

---

## 8. Confidence Scoring

| Source Type | Base Confidence |
|------------|----------------|
| Explicit statement by operator | 0.95 |
| Correction from operator | 1.0 |
| Inferred from context | 0.6 |

**Dynamics:** `+0.05` per repeat mention. `-0.05` per 30 days without mention (floor 0.2). Corrections always reset to 1.0 and get a `1.5x` retrieval boost.

---

## 9. Soul Keeper — Growth Workflow

1. **Snapshot:** Copy current `SOUL.md` to `SOUL_HISTORY/`.
2. **Analyze:** Read recent conversations, memory files, corrections.
3. **Edit:** Make small, evidence-based changes directly to `SOUL.md`.
4. **Log:** Write detailed entry to `CHANGELOG.md`.
5. **Rollback:** If operator requests a revert, copy snapshot back and log.

---

## 10. Extension Points

Configure via `CONFIG.md` to cleanly integrate external specs.

- **`crm_context_budget`**: Tokens reserved for the CRM system. The Assembler checks profiles based on current conversation entities.
- **`external_context_budget` & `external_context_source`**: Tokens and filepath for external context (e.g., Breadcrumb System's `recipe.json`).
- **`recall_external_handler`**: Script path to route non-chronological recall failures (e.g., tool/script amnesia).

---

## 11. Complete Schedule

| Process | Type | Default Time | Day |
|---------|------|-------------|-----|
| Chat Logging | Continuous | — | — |
| Heartbeat Triage | Per message | — | — |
| Recall Validator | Immediate | — | — |
| Soul Keeper | Cron | 02:30 | Daily |
| Memory Miner | Cron | 06:00 | Daily |
| Conflict Detector | Cron | 07:00 | Daily |
| Memory Compactor | Cron | 03:00 | Sunday |
| Vector DB Hygiene | Cron | 03:30 | Sunday |

---

## 12. File Manifest

| File | Owner | Purpose |
|------|-------|---------|
| `SOUL.md` | Soul Keeper | Agent identity |
| `USER.md` | Operator | Baseline operator profile |
| `MEMORY.md` | Memory Miner | Curated long-term chronological memory |
| `CONFIG.md` | Operator | Tunable parameters |
| `CORRECTIONS.md` | Recall Validator | Recent recall failures |
| `CONFLICTS.md` | Conflict Detector | Detected contradictions |
| `CHANGELOG.md` | Soul Keeper | Identity evolution log |
| `COMPACTION_LOG.md` | Memory Compactor | Archival audit trail |

---

## 13. CONFIG.md Template

```yaml
# ── Context Assembler ──
context_budget_tokens: 6000
identity_tokens: 800
user_profile_tokens: 600
corrections_tokens: 400
recent_memory_tokens: 1200
semantic_recall_tokens: 1500
crm_tokens: 1000
longterm_memory_tokens: 500

# ── Memory ──
memory_retention_hours: 48
compaction_age_days: 30
correction_archive_days: 14
correction_overflow_strategy: "prioritize_unverified"
corrected_embedding_grace_days: 7
confidence_decay_interval_days: 30
confidence_decay_amount: 0.05
confidence_floor: 0.2
confidence_boost_per_mention: 0.05
correction_relevance_boost: 1.5

# ── Soul Keeper ──
soul_max_tokens: 800
soul_history_retention_days: 90

# ── Schedule ──
cron_soul_keeper: "02:30"
cron_memory_miner: "06:00"
cron_conflict_detector: "07:00"
cron_compactor_weekly: "Sunday 03:00"

# ── Extension Points ──
external_context_budget: 0
external_context_source: ""
recall_external_handler: ""
heartbeat_custom_patterns: []
```

---

## 14. Build Order

1. **Directories:** Create `memory/`, `Archive/Chronicles/`, `Archive/Corrections/`, `SOUL_HISTORY/`, and agent folders (`agents/memory-miner`, etc.).
2. **Config:** Deploy `CONFIG.md` and adjust token budgets.
3. **Core Files:** Touch `MEMORY.md`, `CORRECTIONS.md`, `CONFLICTS.md`, `CHANGELOG.md`, `COMPACTION_LOG.md`. Create initial `SOUL.md` and `USER.md`.
4. **Deploy Agents:** Place the `SOUL.md` system prompts into their respective agent folders.
5. **Heartbeat:** Wire regex triggers to the OpenClaw triage engine.
6. **Schedule:** Add agent execution to system crontab.