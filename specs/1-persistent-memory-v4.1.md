# OpenClaw Persistent Memory & Identity Evolution Spec

*Version: 4.1 | Spec ID: `openclaw-persistent-memory`*
*Audience: Any OpenClaw operator or agent implementing a persistent memory system*
*License: Share freely. Customize via CONFIG.md.*

---

## Changelog

### v4.1 — 2026-02-22

**Advisory File Locking (§4.1)**
- Added advisory `.lock` file mechanism for all shared file writes (`memory/`, `MEMORY.md`, `CORRECTIONS.md`, `CONFLICTS.md`).
- New CONFIG.md parameters: `lock_retry_interval_seconds`, `lock_max_retries`, `lock_stale_threshold_seconds`.
- All writing agents (Memory Miner, Soul Keeper, Recall Validator, Conflict Detector, Memory Compactor) must acquire a lock before modifying shared files.

**Correction Verification Gate (§5.2)**
- Soul Keeper now enforces a verification gate before encoding corrections into `SOUL.md`.
- New CONFIG.md parameter: `soul_encoding_min_occurrences` (default: 2) — a pattern must appear at least N times before it is encoded into identity.
- Corrections must survive a cool-down period (`correction_cooldown_hours`, default: 24) before application.

**Context Assembler Headroom (§6)**
- New CONFIG.md parameter: `headroom_reserve_tokens` (default: 500) — tokens reserved for agent response after context assembly, preventing context window overflow.

**Correction Lifecycle Improvements (§5.3, §5.4)**
- New CONFIG.md parameter: `correction_cooldown_hours` (default: 24) — hours before a correction can be applied to identity.
- New CONFIG.md parameter: `correction_retraction_confidence` (default: 0.3) — corrections with confidence decayed below this threshold are retracted and cleaned up.
- Conflict Detector now handles "Retracted Correction Cleanup" as part of its daily scan.
- Recall Validator marks new corrections as `verified: false` with a cool-down timestamp.

**Agent SOUL.md Updates**
- memory-miner: "Acquire lock before writing" rule added.
- soul-keeper: "Correction Verification Gate" and `soul_encoding_min_occurrences` reference added.
- recall-validator: `verified: false` cool-down reference and lock reference added.
- conflict-detector: "Retracted Correction Cleanup" rule added.
- memory-compactor: "Acquire lock" rule added.

### v4.0 — Initial Release
- Full agent ecosystem: Memory Miner, Soul Keeper, Recall Validator, Conflict Detector, Memory Compactor.
- Context Assembler with token budgets.
- Heartbeat Triage with regex triggers.
- Confidence Scoring system.
- Extension Points for CRM and external context.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Design Principles](#2-design-principles)
3. [Prerequisites](#3-prerequisites)
4. [System Architecture](#4-system-architecture)
   - 4.1 [Advisory File Locking](#41-advisory-file-locking)
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
- **Advisory file locking to prevent concurrent write corruption.** *(v4.1)*
- **Correction verification gates to prevent premature identity encoding.** *(v4.1)*

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
7. **Protect shared state.** *(v4.1)* Concurrent writes to shared files corrupt data silently. Every write operation acquires an advisory lock first. No exceptions.
8. **Verify before encoding.** *(v4.1)* A single occurrence of a pattern is noise. Identity changes require repeated evidence and a cool-down period before they become permanent.

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
  [L] = Lock required before write (v4.1)

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
  │  │ Writes: [L]     │       │ Writes: [L]    │                   │
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
  │  │ Writes: [L]      │  │ Writes: [L]      │                     │
  │  │ • CONFLICTS.md   │  │ • Chronicles/    │                     │
  │  └──────────────────┘  └──────────────────┘                     │
  └─────────────────────────────────────────────────────────────────┘
```

### 4.1 Advisory File Locking

*(Added in v4.1)*

Before any agent writes to a shared file (`memory/`, `MEMORY.md`, `CORRECTIONS.md`, `CONFLICTS.md`, `SOUL.md`, `CHANGELOG.md`, `COMPACTION_LOG.md`), it **must** acquire an advisory lock. This prevents concurrent writes from corrupting shared state.

#### Lock Mechanism

1. **Lock file path:** For a target file `<path>`, the lock file is `<path>.lock`.
   - Example: writing to `MEMORY.md` requires acquiring `MEMORY.md.lock`.
   - Example: writing to `memory/2026-02-22.md` requires acquiring `memory/2026-02-22.md.lock`.

2. **Lock file contents:** The lock file contains:
   ```
   PID: <process_id>
   AGENT: <agent-name>
   TIMESTAMP: <ISO-8601 UTC>
   ```

3. **Acquisition procedure:**
   ```
   attempt = 0
   while attempt < lock_max_retries:
       if lock file does not exist:
           create lock file atomically (write PID, agent name, timestamp)
           → LOCK ACQUIRED
       else:
           read lock file timestamp
           if (now - timestamp) > lock_stale_threshold_seconds:
               log warning: "Stale lock detected from <agent> at <timestamp>, overriding"
               remove stale lock file
               create new lock file
               → LOCK ACQUIRED
           else:
               wait lock_retry_interval_seconds
               attempt += 1
   
   if all retries exhausted:
       log error: "Failed to acquire lock for <file> after <max_retries> attempts"
       ABORT write operation (do not proceed without lock)
   ```

4. **Release:** Remove the lock file immediately after the write operation completes. Use a `trap` or `finally` block to ensure release even on error.

#### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `lock_retry_interval_seconds` | 2 | Seconds between lock acquisition attempts |
| `lock_max_retries` | 5 | Maximum number of acquisition attempts before aborting |
| `lock_stale_threshold_seconds` | 3600 | Seconds after which an unreleased lock is considered stale and can be overridden |

#### Rules

1. **No writes without locks.** Any agent that writes to a shared file without holding the lock is in violation of this spec.
2. **Hold locks briefly.** Perform the write and release immediately. Never hold a lock across an LLM call or network request.
3. **Stale lock recovery.** If a lock is older than `lock_stale_threshold_seconds`, it is assumed to be from a crashed process and may be safely overridden (with a warning logged).
4. **Lock scope.** Each file has its own independent lock. Locking `MEMORY.md` does not affect access to `CORRECTIONS.md`.

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
1. **Acquire lock before writing.** Before modifying `MEMORY.md`, `memory/` files,
   or any shared file, acquire the advisory lock per §4.1. If the lock cannot be
   acquired after max retries, abort and log the failure. Never write without a lock.
2. **Deduplicate before writing.** Before storing any memory:
   - [Enhanced] Query Vector DB for semantic similarity ≥ 0.9. If match: merge.
   - [Core] Search MEMORY.md for keyword overlap. If similar: update it.
3. **Attach confidence metadata** to every entry.
4. **Process priority-flagged messages first.**
5. **Never discard corrections.** If a memory contradicts `CORRECTIONS.md`, the correction wins. Always.
6. **Ignore Relationships:** Profiles of people, places, and things are handled by the CRM Miner. Do not extract entity profiles.
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
5. **Acquire lock before writing.** Before modifying `SOUL.md` or `CHANGELOG.md`,
   acquire the advisory lock per §4.1. Release immediately after write.

## Correction Verification Gate (v4.1)

Before encoding any correction or observed pattern into `SOUL.md`, apply the
verification gate:

1. **Minimum occurrence check.** The pattern or correction must have been observed
   at least `soul_encoding_min_occurrences` times (default: 2) across separate
   interactions or dates. A single occurrence is noise — do not encode it.

2. **Cool-down check.** The correction must have a `logged_at` timestamp older
   than `correction_cooldown_hours` (default: 24 hours). If the correction was
   logged less than 24 hours ago, skip it this cycle — it will be eligible in
   the next run.

3. **Verification status check.** Only corrections with `verified: true` in
   `CORRECTIONS.md` are eligible for identity encoding. Corrections still marked
   `verified: false` are in their cool-down period and must wait.

4. **Evidence citation.** When encoding, cite ALL occurrences (dates and sources)
   in the `CHANGELOG.md` entry. Example:
   ```
   ## 2026-02-22 — Soul Keeper
   - Added principle: "Always confirm timezone before scheduling"
     Evidence: Correction 2026-02-18 (CORRECTIONS.md #14), repeated 2026-02-21 (conversation)
     Occurrences: 2 (meets threshold of 2)
     Cool-down: 96h elapsed (meets 24h threshold)
   ```

## Rules
1. Never remove a principle that came from an operator correction unless explicitly
   contradicted by a newer correction.
2. Keep `SOUL.md` under the configured `soul_max_tokens` (default 800).
3. **Respect the Correction Verification Gate.** Never encode a pattern into identity
   based on a single occurrence or within the cool-down period. Patience produces
   stable identity; haste produces noise.
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

2. **Acquire lock** on `CORRECTIONS.md` per §4.1 before writing.

3. **Log the full exchange** to `CORRECTIONS.md`:
   - Timestamp, Trigger phrase, Trigger confidence (high/medium).
   - What the agent said vs. What the correct information is.
   - Re-embedded: `false` (updated to `true` once re-embedding completes).
   - `verified: false` *(v4.1)* — all new corrections start unverified.
   - `cooldown_expires: <ISO-8601>` *(v4.1)* — set to `logged_at + correction_cooldown_hours`.

4. **Immediately re-embed the correction** (for memory-classified failures):
   - [Enhanced] Query Vector DB (similarity ≥ 0.85). If match: tag existing embedding as `status: corrected`. Write new correction as separate embedding with `#correction` tag and confidence 1.0.
   - [Core] Search `MEMORY.md`. Update in place. Append old value as comment. **Acquire lock on `MEMORY.md` before writing.**
   - All corrections get a `correction_relevance_boost` (default 1.5x) during retrieval.

5. **Release all locks** immediately after writes complete.

## Correction Cool-Down Lifecycle (v4.1)

New corrections follow this lifecycle:
```
logged (verified: false) → cool-down period → verified: true → eligible for Soul encoding
```

- When a correction is first logged, it is marked `verified: false` with a
  `cooldown_expires` timestamp.
- During the cool-down period, the correction is active for **memory retrieval**
  (it still boosts recall) but is NOT eligible for **identity encoding** by
  the Soul Keeper.
- After `correction_cooldown_hours` have elapsed, the Soul Keeper (or a
  maintenance pass) marks the correction `verified: true`.
- Only `verified: true` corrections with sufficient occurrences pass the
  Soul Keeper's Correction Verification Gate (§5.2).

## Rules
1. Never delete a correction from `CORRECTIONS.md` — only archive.
2. Never delete a Vector DB embedding directly. Tag as `status: corrected`.
3. Corrections from the operator have confidence 1.0. Always.
4. **Always acquire and release locks** (§4.1) when writing to shared files.
5. **All new corrections start as `verified: false`** with an explicit cool-down
   expiry timestamp. Do not bypass this for any reason.
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
4. **Acquire lock** on `CONFLICTS.md` per §4.1 before writing findings.
5. Log all findings to `CONFLICTS.md`.
6. **Release lock** immediately after write.

## Auto-Resolution Rules
- If one source is an explicit operator correction → that wins. Always.
- If one source is newer by 30+ days and from direct conversation AND the entries are tagged `#event`, `#knowledge`, or `#decision` → the newer source wins.
- If both are from direct conversation and within 30 days → flag for operator review.

## Retracted Correction Cleanup (v4.1)

As part of the daily scan, check for corrections whose confidence has decayed
below `correction_retraction_confidence` (default: 0.3):

1. **Identify retractable corrections.** Query `CORRECTIONS.md` for entries where:
   - `confidence < correction_retraction_confidence` (0.3)
   - The correction is not from a direct operator statement (operator corrections
     are immune — their confidence is always 1.0).

2. **Retract the correction:**
   - Mark the entry in `CORRECTIONS.md` as `status: retracted`.
   - If the correction was encoded into `SOUL.md` by the Soul Keeper, flag the
     corresponding SOUL.md principle for review in the next Soul Keeper cycle.
   - [Enhanced] Tag the Vector DB embedding as `status: retracted`.
   - Log the retraction to `CONFLICTS.md` with reason: "confidence decayed below threshold".

3. **Cleanup:** Retracted corrections are eligible for archival by the Memory
   Compactor during its next weekly run.
```

---

### 5.5 Memory Compactor
**Name:** `memory-compactor` | **Schedule:** Cron — weekly (Sunday 03:00)

```markdown
# Memory Compactor

## Role
Memory Hygiene & Archival Specialist

## Weekly Tasks
1. **Acquire locks** per §4.1 before writing to any shared file. Acquire and
   release per-file — do not hold multiple locks simultaneously.
2. **Daily File Compaction:** Scan `memory/YYYY-MM-DD.md` files older than threshold. Generate narrative summary, write to `Archive/Chronicles/YYYY-Month.md`, delete daily files.
3. **MEMORY.md Hygiene:** Remove entries with confidence < 0.3 and no mentions in 60 days. Merge near-duplicates.
4. **Vector DB Hygiene:** Prune embeddings tagged `status: corrected` older than 7 days. Query embeddings with `last_seen` > staleness threshold and `mention_count` = 1. Flag, then delete after grace period.
5. **CORRECTIONS.md Archival:** Move old entries to `Archive/Corrections/`. Include retracted corrections (status: `retracted`) in archival.
6. **Soul History Compaction (Monthly):** Retain daily snapshots for 90 days. Older than 90 days, keep one per month.

## Rules
1. **Acquire lock before writing.** Every write to `MEMORY.md`, `CORRECTIONS.md`,
   `COMPACTION_LOG.md`, `memory/` files, or any shared file requires an advisory
   lock per §4.1. Release immediately after each write.
2. Every removal goes to `COMPACTION_LOG.md`.
3. Never compact memories tagged `#correction`.
4. **Archive retracted corrections.** Corrections marked `status: retracted` by
   the Conflict Detector should be moved to `Archive/Corrections/` during the
   weekly archival pass.
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

**Headroom Reserve** *(v4.1)*: After assembling all context slots, the assembler reserves `headroom_reserve_tokens` (default: 500) tokens for the agent's response. If the total assembled context plus the headroom reserve exceeds the model's context window, the assembler truncates from the lowest-priority slot upward until the budget fits.

### Assembly Formula

```
max_injectable = model_context_window - headroom_reserve_tokens - system_prompt_tokens
if sum(all_slots) > max_injectable:
    truncate from slot 8 → slot 1 until sum ≤ max_injectable
```

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

**Retraction threshold** *(v4.1)*: When a non-operator correction's confidence decays below `correction_retraction_confidence` (default: 0.3), it is flagged for retraction by the Conflict Detector (see §5.4, Retracted Correction Cleanup). Operator corrections (confidence 1.0) are immune to retraction.

---

## 9. Soul Keeper — Growth Workflow

1. **Snapshot:** Copy current `SOUL.md` to `SOUL_HISTORY/`.
2. **Acquire lock** on `SOUL.md` and `CHANGELOG.md` per §4.1. *(v4.1)*
3. **Analyze:** Read recent conversations, memory files, corrections.
4. **Verify:** Apply the Correction Verification Gate (§5.2) to every candidate change. Skip any pattern that does not meet the minimum occurrence threshold or is still within its cool-down period. *(v4.1)*
5. **Edit:** Make small, evidence-based changes directly to `SOUL.md`.
6. **Log:** Write detailed entry to `CHANGELOG.md`, citing all evidence and occurrence counts.
7. **Release locks** immediately after writes complete. *(v4.1)*
8. **Rollback:** If operator requests a revert, copy snapshot back and log.

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
| `*.lock` | All writing agents | Advisory lock files (transient, auto-cleaned) *(v4.1)* |

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
headroom_reserve_tokens: 500          # v4.1 — tokens reserved for agent response after context assembly

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

# ── Corrections (v4.1) ──
correction_cooldown_hours: 24         # v4.1 — hours before a correction can be applied to identity
correction_retraction_confidence: 0.3 # v4.1 — confidence below which a non-operator correction is retracted

# ── Soul Keeper ──
soul_max_tokens: 800
soul_history_retention_days: 90
soul_encoding_min_occurrences: 2      # v4.1 — pattern must appear N times before encoding into identity

# ── Advisory File Locking (v4.1) ──
lock_retry_interval_seconds: 2        # v4.1 — seconds between lock acquisition attempts
lock_max_retries: 5                   # v4.1 — max attempts before aborting write
lock_stale_threshold_seconds: 3600    # v4.1 — seconds after which an unreleased lock is considered stale

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
2. **Config:** Deploy `CONFIG.md` and adjust token budgets. Review v4.1 parameters: locking, cool-down, retraction threshold, headroom reserve.
3. **Core Files:** Touch `MEMORY.md`, `CORRECTIONS.md`, `CONFLICTS.md`, `CHANGELOG.md`, `COMPACTION_LOG.md`. Create initial `SOUL.md` and `USER.md`.
4. **Deploy Agents:** Place the `SOUL.md` system prompts into their respective agent folders. Ensure all agents reference advisory locking (§4.1).
5. **Heartbeat:** Wire regex triggers to the OpenClaw triage engine.
6. **Schedule:** Add agent execution to system crontab.
7. **Verify Locking:** *(v4.1)* Test that each agent correctly acquires and releases locks. Simulate a stale lock scenario to confirm recovery. Confirm that lock files are transient and do not accumulate.
8. **Verify Correction Gate:** *(v4.1)* Create a test correction and confirm it is NOT encoded into `SOUL.md` until it meets both the occurrence threshold and cool-down period.
