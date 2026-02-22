# OpenClaw Deployment Auditor — Implementation Verification Spec

*Version: 1.0 | Spec ID: `openclaw-deployment-auditor`*
*Audience: OpenClaw operators who have deployed one or more infrastructure specs and need to verify the implementation is complete and correct.*
*License: Share freely.*

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Design Principles](#2-design-principles)
3. [Prerequisites](#3-prerequisites)
4. [Agent Definition](#4-agent-definition)
5. [Audit Scope & Tier Detection](#5-audit-scope--tier-detection)
6. [Audit Checklists](#6-audit-checklists)
   - 6.1 [Persistent Memory Audit](#61-persistent-memory-audit)
   - 6.2 [Breadcrumb System Audit](#62-breadcrumb-system-audit)
   - 6.3 [CRM System Audit](#63-crm-system-audit)
   - 6.4 [Integration Bridge Audit](#64-integration-bridge-audit)
   - 6.5 [Lab Manager Audit](#65-lab-manager-audit)
   - 6.6 [Cross-System Audit](#66-cross-system-audit)
7. [Audit Report Format](#7-audit-report-format)
8. [Validation Tests](#8-validation-tests)
9. [Common Deployment Failures](#9-common-deployment-failures)
10. [Schedule & Invocation](#10-schedule--invocation)
11. [Build Order](#11-build-order)

---

## 1. Purpose

### The Problem

When an operator hands a spec to an agent and says "implement this," the agent will:

- Interpret ambiguous sections in its own favor (usually: skip them).
- Implement required components but skip optional/enhanced components — even when the infrastructure for them exists.
- Create files and directories but skip the wiring (cron entries, heartbeat patterns, CONFIG.md parameters).
- Say "done" without running any of the validation steps defined in the spec's Build Order.
- Make undocumented architectural decisions (different file paths, renamed parameters, omitted fields).

The operator has no way to know what was actually done versus what should have been done without manually auditing every file, every cron entry, and every CONFIG.md parameter against the spec.

### The Solution

The Deployment Auditor is an agent that reads the specs, reads the filesystem, and produces a gap analysis. It tells the operator exactly:

- What was implemented correctly.
- What was implemented incorrectly (wrong path, wrong format, missing fields).
- What was skipped entirely.
- What optional/enhanced components could have been implemented given the available infrastructure.
- Which validation tests from the Build Orders have been run — and which haven't.

### The Distinction: Auditor vs. Lab Manager

| Concern | Lab Manager | Deployment Auditor |
|---------|------------|-------------------|
| **Core question** | "Is the running system healthy?" | "Was the system built correctly?" |
| **When to use** | Ongoing operations (weekly/daily) | Post-deployment, post-upgrade, on suspicion |
| **What it catches** | Runtime degradation, metric drift | Missing implementations, skipped wiring, wrong configs |
| **Example finding** | "Baker hasn't run in 3 days" | "Baker was never added to crontab" |
| **Tier awareness** | Reports on what's present | Identifies what's present AND what's missing given available infra |
| **Spec awareness** | Reads logs and artifacts | Reads specs and compares against actual implementation |

The Auditor runs first (to verify the build). The Lab Manager runs continuously (to verify operations). They are complementary, not redundant.

---

## 2. Design Principles

1. **Spec is the source of truth.** The Auditor compares the implementation against the specification, not against assumptions or best practices. If the spec says it, it should exist.
2. **Detect the tier, audit the tier.** If Enhanced infrastructure (Vector DB, messaging channel) is available, the Auditor checks that Enhanced-tier features were actually wired up — not just the Core tier.
3. **Silent on opinions, loud on gaps.** The Auditor does not recommend architectural changes. It reports what the spec requires and whether it was done. Recommendations belong to the Lab Manager.
4. **Every finding is actionable.** Each gap includes the specific file, path, config parameter, or cron entry that needs to be created or fixed — not just "this is missing."
5. **Read-only.** The Auditor never creates, modifies, or deletes any file. It only reads and reports. The operator or implementing agent acts on the findings.
6. **Idempotent.** Running the Auditor multiple times produces the same report if nothing has changed. It has no side effects.
7. **Run on suspicion.** The Auditor is not scheduled. It runs when the operator suspects something is wrong, after a deployment, or after an upgrade. It can optionally be scheduled for periodic compliance checks.

---

## 3. Prerequisites

### Required

| Component | Purpose | Notes |
|-----------|---------|-------|
| OpenClaw instance | Agent runtime | Any supported platform |
| At least one deployed spec | Something to audit | Auditor detects which specs are deployed |
| Spec documents | The canonical reference | Auditor needs access to the spec `.md` files |
| `jq` | JSON validation | Standard Linux utility |

### Spec Document Locations

The Auditor expects spec documents at a known location. Default: `~/.openclaw/workspace/specs/`. The operator should place the spec markdown files here (or configure the path in CONFIG.md).

```yaml
# CONFIG.md addition
auditor_spec_directory: "~/.openclaw/workspace/specs/"
```

If the spec directory is not found, the Auditor falls back to its embedded checklist knowledge (this document). However, the spec files are preferred because they may contain operator-customized parameters that override defaults.

---

## 4. Agent Definition

**Name:** `deployment-auditor`
**Location:** `agents/deployment-auditor/`
**Schedule:** On-demand (default). Optional: periodic (e.g., monthly or post-upgrade).

### SOUL.md

```markdown
# Deployment Auditor

## Role
Implementation Verification Specialist

## Mission
Verify that deployed OpenClaw infrastructure specs have been implemented
completely and correctly. Produce a gap analysis that tells the operator
exactly what was done, what was missed, and what needs to be fixed.

You are not a health monitor (that's the Lab Manager). You are a building
inspector. You compare the blueprints (specs) against the building
(implementation) and report every deviation.

## How You Work

### Phase 1: DETECT (What's deployed? What tier?)

1. Check for the existence of each spec's signature files:
   - Persistent Memory: MEMORY.md, SOUL.md, CORRECTIONS.md
   - Breadcrumb System: ~/breadcrumb-trail/recipe.json, ~/breadcrumb-trail/index.json
   - CRM System: Archive/People/, Archive/Places/, Archive/Things/
   - Integration Bridge: CONFIG.md with external_context_budget > 0
   - Lab Manager: ~/lab-reports/

2. Detect available Enhanced infrastructure:
   - Vector DB: Check for vector DB connection (test query or config presence)
   - Messaging channel: Check CONFIG.md for messaging configuration
   - NFS/shared storage: Check for mount points in CONFIG.md

3. Determine the expected tier for each deployed spec:
   - If Enhanced infrastructure is available → spec SHOULD be at Enhanced tier
   - Compare against what's actually wired

### Phase 2: AUDIT (Compare spec against implementation)

For each detected spec, walk through its complete checklist (Section 6).
For each item:

- **EXISTS**: The file, directory, config parameter, cron entry, or script exists.
- **CORRECT**: The content matches the spec's requirements (format, fields, values).
- **WIRED**: The component is connected to the rest of the system (cron runs it,
  heartbeat routes to it, CONFIG.md references it).

Record the result as one of:
- ✅ PASS — Exists, correct, and wired.
- ⚠️ PARTIAL — Exists but has issues (wrong format, missing fields, not wired).
- ❌ MISSING — Does not exist.
- 🔵 SKIPPED (OPTIONAL) — Optional component not implemented; Enhanced infra
  not available. Acceptable.
- 🟡 SKIPPED (AVAILABLE) — Optional component not implemented but Enhanced
  infra IS available. Should have been implemented.

### Phase 3: VALIDATE (Run Build Order tests)

Execute the validation tests from each spec's Build Order section.
Record pass/fail for each test.

### Phase 4: REPORT (Generate gap analysis)

Produce a structured audit report with:
- System-by-system findings
- Tier analysis (what tier is deployed vs. what tier is possible)
- Prioritized fix list
- Validation test results

## Rules
1. NEVER modify any file. You are strictly read-only.
2. NEVER make architectural recommendations. Report gaps, not opinions.
3. Every finding must reference the specific spec section that defines the requirement.
4. Distinguish between "missing because the operator chose not to implement"
   and "missing because the implementing agent skipped it."
5. If you cannot determine whether a component is correctly configured (e.g.,
   you can't test the Vector DB connection), report it as "UNABLE TO VERIFY"
   with the reason.
6. Be specific. "CORRECTIONS.md is missing the verified field" is useful.
   "Memory system has issues" is not.
```

---

## 5. Audit Scope & Tier Detection

### Tier Detection Logic

The Auditor must determine what the operator's environment is capable of, not just what's currently deployed. This is the key insight: if an agent implements only Core tier when Enhanced infrastructure exists, that's a gap.

```
TIER DETECTION PROCEDURE:

1. Vector DB available?
   - Check: CONFIG.md contains vector_db_* parameters with non-empty values
   - Check: psql or equivalent can reach the configured vector store
   - Check: Any .env or secrets file references a vector DB connection string
   - Result: If ANY check passes → Enhanced tier EXPECTED for memory operations

2. Messaging channel available?
   - Check: CONFIG.md contains messaging_* parameters
   - Check: Any webhook URL or bot token is configured
   - Result: If ANY check passes → Enhanced tier EXPECTED for notifications

3. NFS / shared storage available?
   - Check: CONFIG.md contains mount definitions
   - Check: /mnt/ or configured mount paths are accessible
   - Result: If check passes → mount-aware breadcrumbs EXPECTED
```

### Spec Detection Logic

```
SPEC DETECTION PROCEDURE:

Persistent Memory deployed if ALL exist:
  - SOUL.md (non-empty)
  - USER.md (non-empty)
  - memory/ directory
  - CONFIG.md with memory-related parameters

Breadcrumb System deployed if ALL exist:
  - ~/breadcrumb-trail/ directory
  - ~/breadcrumb-trail/recipe.json
  - ~/breadcrumb-trail/index.json

CRM System deployed if ALL exist:
  - Archive/People/ directory
  - Archive/Places/ directory
  - Archive/Things/ directory

Integration Bridge active if:
  - CONFIG.md contains external_context_budget > 0
  - OR CONFIG.md contains external_context_source with a non-empty value
  - OR CONFIG.md contains crm_entity_exclusion_source with a non-empty value

Lab Manager deployed if:
  - ~/lab-reports/ directory exists
  - agents/lab-manager/ directory exists
```

---

## 6. Audit Checklists

Each checklist is organized by the spec's Build Order. Items marked `[E]` are Enhanced-tier only. Items marked `[OPT]` are explicitly optional regardless of tier.

### 6.1 Persistent Memory Audit

#### Directories

| Check | Path | Spec Reference |
|-------|------|---------------|
| memory/ exists | `~/.openclaw/workspace/memory/` | §14 Build Order, Step 1 |
| Archive/Chronicles/ exists | `Archive/Chronicles/` | §14 Build Order, Step 1 |
| Archive/Corrections/ exists | `Archive/Corrections/` | §14 Build Order, Step 1 |
| SOUL_HISTORY/ exists | `SOUL_HISTORY/` | §14 Build Order, Step 1 |
| agents/memory-miner/ exists | `agents/memory-miner/` | §14 Build Order, Step 4 |
| agents/soul-keeper/ exists | `agents/soul-keeper/` | §14 Build Order, Step 4 |
| agents/recall-validator/ exists | `agents/recall-validator/` | §14 Build Order, Step 4 |
| agents/conflict-detector/ exists | `agents/conflict-detector/` | §14 Build Order, Step 4 |
| agents/memory-compactor/ exists | `agents/memory-compactor/` | §14 Build Order, Step 4 |

#### Core Files

| Check | Validation | Spec Reference |
|-------|-----------|---------------|
| SOUL.md exists and non-empty | `test -s SOUL.md` | §14 Step 3 |
| USER.md exists and non-empty | `test -s USER.md` | §14 Step 3 |
| MEMORY.md exists | `test -f MEMORY.md` | §14 Step 3 |
| CORRECTIONS.md exists | `test -f CORRECTIONS.md` | §14 Step 3 |
| CONFLICTS.md exists | `test -f CONFLICTS.md` | §14 Step 3 |
| CHANGELOG.md exists | `test -f CHANGELOG.md` | §14 Step 3 |
| COMPACTION_LOG.md exists | `test -f COMPACTION_LOG.md` | §14 Step 3 |
| CONFIG.md exists and non-empty | `test -s CONFIG.md` | §14 Step 2 |

#### CONFIG.md Parameters

| Parameter | Expected | Validation | Spec Reference |
|-----------|----------|-----------|---------------|
| `context_budget_tokens` | ≥ 5000 | Numeric, present | §13 |
| `headroom_reserve_tokens` | > 0 | Numeric, present | §6 (v4.1) |
| `identity_tokens` | > 0 | Numeric | §13 |
| `user_profile_tokens` | > 0 | Numeric | §13 |
| `corrections_tokens` | > 0 | Numeric | §13 |
| `recent_memory_tokens` | > 0 | Numeric | §13 |
| `semantic_recall_tokens` | > 0 | Numeric | §13 |
| `longterm_memory_tokens` | > 0 | Numeric | §13 |
| `correction_cooldown_hours` | > 0 | Numeric, present | §8 (v4.1) |
| `correction_retraction_confidence` | 0-1 | Numeric, present | §8 (v4.1) |
| `soul_encoding_min_occurrences` | ≥ 2 | Numeric, present | §5.2 (v4.1) |
| `lock_retry_interval_seconds` | > 0 | Numeric, present | §4.1 (v4.1) |
| `lock_max_retries` | > 0 | Numeric, present | §4.1 (v4.1) |
| `lock_stale_threshold_seconds` | > 0 | Numeric, present | §4.1 (v4.1) |
| **Budget sum check** | Sum of slot budgets ≤ `context_budget_tokens` | Arithmetic | §6 |

#### Agent SOUL.md Content Checks

| Agent | Key Content Check | Spec Reference |
|-------|------------------|---------------|
| memory-miner | Contains "Acquire lock before writing" | §5.1 (v4.1) |
| soul-keeper | Contains "Correction Verification Gate" | §5.2 (v4.1) |
| soul-keeper | Contains "soul_encoding_min_occurrences" or "2 occurrences" | §5.2 (v4.1) |
| recall-validator | Contains "verified: false" (cool-down reference) | §5.3 (v4.1) |
| recall-validator | Contains "lock" reference | §5.3 (v4.1) |
| conflict-detector | Contains "Retracted Correction Cleanup" | §5.4 (v4.1) |
| memory-compactor | Contains "Acquire lock" | §5.5 (v4.1) |

#### Cron Schedule

| Job | Expected Entry | Validation |
|-----|---------------|-----------|
| Soul Keeper | `30 2 * * *` (or CONFIG value) | `crontab -l \| grep soul-keeper` |
| Memory Miner | `0 6 * * *` (or CONFIG value) | `crontab -l \| grep memory-miner` |
| Conflict Detector | `0 7 * * *` (or CONFIG value) | `crontab -l \| grep conflict-detector` |
| Memory Compactor | `0 3 * * 0` (Sunday) | `crontab -l \| grep memory-compactor` |

#### Heartbeat Wiring

| Pattern Group | Patterns Registered | Validation |
|--------------|-------------------|-----------|
| Memory triggers | 7 patterns (§7) | Check heartbeat config for all 7 |
| Recall failure triggers | 8 patterns (§7) | Check heartbeat config for all 8 |
| Soft correction triggers | 5 patterns (§7) | Check heartbeat config for all 5 |

#### Enhanced Tier `[E]`

| Check | Validation | Spec Reference |
|-------|-----------|---------------|
| Vector DB connection configured | CONFIG.md or .env has connection params | §3 Enhanced |
| Vector DB is reachable | Test query succeeds | §3 Enhanced |
| Memory Miner writes to Vector DB | SOUL.md mentions Vector DB protocol | §5.1 |
| Recall Validator queries Vector DB | SOUL.md mentions similarity ≥ 0.85 | §5.3 |
| Conflict Detector uses Vector DB | SOUL.md mentions similarity ≥ 0.8 | §5.4 |
| Memory Compactor prunes Vector DB | SOUL.md mentions Vector DB Hygiene | §5.5 |
| Messaging channel configured | CONFIG.md has messaging params | §3 Enhanced |

---

### 6.2 Breadcrumb System Audit

#### Directories

| Check | Path | Spec Reference |
|-------|------|---------------|
| ~/breadcrumb-trail/ exists | `~/breadcrumb-trail/` | §16 Step 1 |
| ~/breadcrumb-trail/orphans/ exists | `~/breadcrumb-trail/orphans/` | §16 Step 1 |
| ~/breadcrumb-trail/archive/ exists | `~/breadcrumb-trail/archive/` | §16 Step 1 |
| agents/breadcrumb-baker/ exists | `agents/breadcrumb-baker/` | §16 Step 7 |

#### Core Files

| Check | Validation | Spec Reference |
|-------|-----------|---------------|
| recipe.json exists and valid JSON | `jq . recipe.json` | §16 Step 3 |
| recipe.json has `capabilities` key | `jq '.capabilities' recipe.json` | §9 |
| recipe.json has `mounts` key | `jq '.mounts' recipe.json` | §9 |
| index.json exists and valid JSON | `jq . index.json` | §16 Step 4 |
| index.json has `breadcrumbs` array | `jq '.breadcrumbs' index.json` | §10 |
| index.json has `tags_index` object | `jq '.tags_index' index.json` | §10 |
| trail.md exists | `test -f trail.md` | §10 |
| schema.crumb.md exists | `test -f schema.crumb.md` | §16 Step 2 |
| baker.log exists | `test -f baker.log` | §12 |
| index.json.bak exists | `test -f index.json.bak` | §10.1 (v3.1) |
| recipe.json.bak exists | `test -f recipe.json.bak` | §9.1 (v3.1) |

#### Baker SOUL.md Content Checks

| Key Content Check | Spec Reference |
|------------------|---------------|
| Contains "Atomic Write Protocol" or "atomic" | §9.1 (v3.1) |
| Contains "Orphan Lifecycle" or "30-day" draft expiration | §8.1 (v3.1) |
| Contains "max_orphan_count" | §8.1 (v3.1) |
| Contains all three phases (DISCOVER, VALIDATE, INDEX) | §12 |
| Contains "COLD START TEST" | §12 |

#### Cron Schedule

| Job | Expected Entry | Validation |
|-----|---------------|-----------|
| Baker 05:00 | `0 5 * * *` | `crontab -l \| grep -i baker` |
| Baker 11:00 | `0 11 * * *` | Verify 4 daily entries |
| Baker 17:00 | `0 17 * * *` | |
| Baker 23:00 | `0 23 * * *` | |

#### Standing Directive

| Check | Validation | Spec Reference |
|-------|-----------|---------------|
| All deployed agents contain BREADCRUMB DIRECTIVE | Grep each agent's SOUL.md for "BREADCRUMB DIRECTIVE" or "PRE-FLIGHT CHECK" | §13 |
| Directive includes all 7 rules | Grep for RULE 1 through RULE 7 | §13 |

#### Index Integrity

| Check | Validation | Spec Reference |
|-------|-----------|---------------|
| All indexed crumbs exist on disk | For each entry in index.json, verify file at `path` exists | §10 |
| All .crumb.md files on disk are indexed | `find ~ -name "*.crumb.md"` vs index.json entries | §10 |
| recipe.json capabilities reference valid crumbs | Each `breadcrumb_path` in recipe.json exists | §9 |

---

### 6.3 CRM System Audit

#### Directories

| Check | Path | Spec Reference |
|-------|------|---------------|
| Archive/People/ exists | `Archive/People/` | §10 Build Order |
| Archive/Places/ exists | `Archive/Places/` | §10 Build Order |
| Archive/Things/ exists | `Archive/Things/` | §10 Build Order |
| agents/crm-miner/ exists | `agents/crm-miner/` | §10 Build Order |

#### CRM Miner SOUL.md Content Checks

| Key Content Check | Spec Reference |
|------------------|---------------|
| Contains "Entity Disambiguation Protocol" or "disambiguation" | §4.1 (v2.1) |
| Contains "Entity Merge Workflow" or "merge" | §4.2 (v2.1) |
| Contains "Acquire lock" | §5 (v2.1) |
| Contains "aliases" reference | §4 (v2.1) |
| Contains "exclusion source" | §5 Rule 2 |

#### CONFIG.md Parameters

| Parameter | Expected | Spec Reference |
|-----------|----------|---------------|
| `staleness_threshold_days` | Numeric (default 90) | §6 |
| `crm_entity_exclusion_source` | Path to recipe.json (if Breadcrumbs deployed) | §8 |
| `max_provisional_age_days` | Numeric (default 30) | §10 (v2.1) |

#### Heartbeat Patterns

| Pattern Group | Count | Spec Reference |
|--------------|-------|---------------|
| Original patterns | 6 patterns | §7 |
| v2.1 additions | 7 patterns | §7 (v2.1) |
| **Total expected** | **13 patterns** | Verify all 13 are registered |

#### Cron Schedule

| Job | Expected Entry | Validation |
|-----|---------------|-----------|
| CRM Miner | `30 6 * * *` | `crontab -l \| grep crm-miner` |

#### Profile Format Compliance (sample check)

For each profile in Archive/:

| Field | Required | Validation |
|-------|----------|-----------|
| YAML frontmatter present | Yes | Starts with `---` |
| `id` field | Yes | Matches `{type}_{slug}` pattern |
| `name` field | Yes | Non-empty |
| `type` field | Yes | One of: person, place, thing |
| `confidence` field | Yes | 0.0 to 1.0 |
| `last_verified` field | Yes | Valid date |
| `aliases` field | Recommended (v2.1) | List or empty list |
| `disambiguation` field | If collision exists | String or null |
| Detail lines end with `[verified: ...]` | Yes | Regex check |

---

### 6.4 Integration Bridge Audit

#### Sentinel Directory

| Check | Path | Spec Reference |
|-------|------|---------------|
| cron-sentinels/ exists | `~/.openclaw/workspace/cron-sentinels/` | §2.1 (v2.1) |

#### CONFIG.md — Bridge-Specific Parameters

| Parameter | Expected | Spec Reference |
|-----------|----------|---------------|
| `external_context_source` | Non-empty path | §3 |
| `recall_external_handler` | Non-empty path | §3 |
| `crm_entity_exclusion_source` | Non-empty path (if CRM deployed) | §3 |
| `cron_overrun_wait_minutes` | Numeric > 0 | §2.1 (v2.1) |
| `cron_sentinel_directory` | Valid path | §2.1 (v2.1) |
| `heartbeat_custom_patterns` | At least 1 entry (breadcrumb_triggers) | §3 |

#### CONFIG Override Compliance

| Check | Validation | Spec Reference |
|-------|-----------|---------------|
| No duplicate budget definitions | Memory spec's standalone budgets are commented out or removed | §3 (v2.1) — CONFIG Override Rule |
| Bridge budgets sum correctly | Sum of all slot budgets ≤ `context_budget_tokens` | §3 (v2.1) |

#### baker-reindex.sh

| Check | Validation | Spec Reference |
|-------|-----------|---------------|
| Script exists | `test -f ~/breadcrumb-trail/scripts/baker-reindex.sh` | §5.1 (v2.1) |
| Script is executable | `test -x ~/breadcrumb-trail/scripts/baker-reindex.sh` | §5.1 (v2.1) |
| Script accepts 2+ args | `bash -n baker-reindex.sh` (syntax check) | §5.1 (v2.1) |
| Script outputs REINDEX_RESULT format | Dry-run test with known search term | §5.1 (v2.1) |

#### Cron Schedule Alignment

| Check | Validation | Spec Reference |
|-------|-----------|---------------|
| All agent cron entries match unified schedule | Compare crontab entries against §2 schedule | §2 |
| No overlapping write windows | Verify timing gaps between sequential jobs | §2 |
| Sentinel files are being written | Check cron-sentinels/ for recent files | §2.1 |

---

### 6.5 Lab Manager Audit

#### Directories & Files

| Check | Path | Spec Reference |
|-------|------|---------------|
| ~/lab-reports/ exists | `~/lab-reports/` | §11 Step 1 |
| agents/lab-manager/ exists | `agents/lab-manager/` | §11 Step 2 |
| Lab Manager SOUL.md exists | `agents/lab-manager/SOUL.md` | §11 Step 2 |
| Heartbeat script exists | `agents/lab-manager/lab-manager-heartbeat.sh` | §10.1 (v1.1) |
| Heartbeat script is executable | `test -x` | §11 Step 4 (v1.1) |

#### Cron Schedule

| Job | Expected Entry | Validation |
|-----|---------------|-----------|
| Weekly report | `0 10 * * 0` (Sunday 10:00) | `crontab -l` |
| Daily heartbeat | `0 8 * * *` (Daily 08:00) | `crontab -l` |

#### Lab Manager SOUL.md Content Checks

| Key Content Check | Spec Reference |
|------------------|---------------|
| Contains "indirect/experimental" reference for Standing Directive metrics | §4 Rule 8 (v1.1) |
| Contains "cron-sentinels" reference | §4 (v1.1) |
| Contains "stale .lock files" reference | §4 (v1.1) |
| Contains "correction cool-down" or "verified" metrics | §5 (v1.1) |

#### Breadcrumb Exists (if Breadcrumb System deployed)

| Check | Validation | Spec Reference |
|-------|-----------|---------------|
| agent.lab-manager.crumb.md exists | In agents/lab-manager/ | §11 Step 3 (v1.1) |
| Breadcrumb has correct YAML fields | Parse frontmatter | §11 Step 3 |

---

### 6.6 Cross-System Audit

These checks validate integration points between specs.

| Check | Systems Involved | Validation | Spec Reference |
|-------|-----------------|-----------|---------------|
| Context Assembler loading order matches Bridge spec | Memory + Bridge | Compare CONFIG.md slot priorities against §4 of Bridge | Bridge §4 |
| CRM exclusion source points to valid recipe.json | CRM + Breadcrumb | `test -f $(grep crm_entity_exclusion_source CONFIG.md)` | CRM §8, Bridge §6 |
| Recall external handler points to valid script | Memory + Breadcrumb | `test -x $(grep recall_external_handler CONFIG.md)` | Memory §10, Bridge §5.1 |
| All agents have Breadcrumb Directive in SOUL.md | All agents + Breadcrumb | Grep all agents' SOUL.md files | Breadcrumb §13 |
| All writing agents reference Advisory File Locking | All writing agents + Memory | Grep for "lock" in each SOUL.md | Memory §4.1 |
| Agent breadcrumbs exist for all system agents | All agents + Breadcrumb | Check for .crumb.md in each agent's directory | Bridge §8 Step 6 |
| Budget sum is consistent across CONFIG.md | Memory + Bridge | Only one set of budget values exists | Bridge §3 (v2.1) |
| Heartbeat patterns include all spec groups | Memory + CRM + Bridge | Total pattern count matches expected | All heartbeat sections |

---

## 7. Audit Report Format

```markdown
# OpenClaw Deployment Audit Report
**Date:** YYYY-MM-DD
**Auditor Version:** 1.0
**Specs Audited Against:** Memory v4.1 | Breadcrumb v3.1 | CRM v2.1 | Bridge v2.1 | Lab Manager v1.1

---

## Executive Summary

**Overall Compliance:** N / N checks passed (XX%)
**Tier Deployed:** Core | Enhanced
**Tier Available:** Core | Enhanced
**Tier Gap:** [None | Enhanced features available but not wired]

| Category | ✅ Pass | ⚠️ Partial | ❌ Missing | 🟡 Skipped (Available) |
|----------|--------|-----------|-----------|----------------------|
| Persistent Memory | N | N | N | N |
| Breadcrumb System | N | N | N | N |
| CRM System | N | N | N | N |
| Integration Bridge | N | N | N | N |
| Lab Manager | N | N | N | N |
| Cross-System | N | N | N | N |
| **Total** | **N** | **N** | **N** | **N** |

---

## Tier Analysis

### Infrastructure Detected
| Component | Available | Wired to Specs |
|-----------|-----------|---------------|
| Vector DB | ✅ / ❌ | ✅ / ❌ |
| Messaging Channel | ✅ / ❌ | ✅ / ❌ |
| NFS/Shared Storage | ✅ / ❌ | ✅ / ❌ |

### Tier Gap Details
[If Enhanced infra is available but not wired, list every Enhanced-tier
check that should have been implemented. Each entry includes the spec
section and the specific action needed.]

---

## [System Name] Findings

### ✅ Passing
[Grouped list of all passing checks — brief, scannable]

### ⚠️ Partial Implementation
[Each finding includes:]
- **What exists:** [description]
- **What's wrong:** [specific issue]
- **Fix:** [exact action — file path, parameter, command]
- **Spec Reference:** [section number]

### ❌ Missing
[Each finding includes:]
- **What's missing:** [description]
- **Required by:** [spec section]
- **Fix:** [exact action to create/implement it]

### 🟡 Skipped (Enhanced Available)
[Each finding includes:]
- **What's not implemented:** [description]
- **Infrastructure available:** [what exists that makes this possible]
- **Fix:** [exact action to wire it up]
- **Spec Reference:** [section number]

---

## Validation Test Results

| Test | System | Result | Notes |
|------|--------|--------|-------|
| Context Assembler loads | Memory | ✅ / ❌ | |
| Heartbeat fires on recall | Memory | ✅ / ❌ | |
| Baker runs and indexes | Breadcrumb | ✅ / ❌ | |
| Cold start test | Breadcrumb | ✅ / ❌ / NOT RUN | |
| recipe.json valid | Breadcrumb | ✅ / ❌ | |
| CRM exclusion works | CRM + Bridge | ✅ / ❌ / NOT RUN | |
| Recall routing works | Memory + Bridge | ✅ / ❌ / NOT RUN | |
| Cron schedule clean | Bridge | ✅ / ❌ | |

---

## Prioritized Fix List

### 🔴 Critical (System won't function correctly)
1. [Specific fix with file path and action]
2. ...

### 🟡 Important (System functions but with gaps)
1. [Specific fix with file path and action]
2. ...

### 🔵 Enhancement (Available infra not utilized)
1. [Specific fix with file path and action]
2. ...

---

## Raw Check Log

[Complete list of every check performed, its result, and the validation
command used. This is the evidence trail.]

---

*Audit completed by Deployment Auditor v1.0 | Specs directory: [path]*
```

---

## 8. Validation Tests

The Auditor runs these tests in Phase 3. Unlike the checklist (which checks static file state), these tests verify dynamic behavior.

### Static Validation (can always run)

| Test | Method | Pass Condition |
|------|--------|---------------|
| CONFIG.md budget arithmetic | Parse and sum all *_tokens values | Sum ≤ context_budget_tokens |
| recipe.json capability→crumb resolution | For each capability, check breadcrumb_path exists | All paths resolve |
| index.json↔filesystem sync | Compare index entries against `find ~ -name "*.crumb.md"` | Bidirectional match |
| Crontab completeness | `crontab -l` vs expected entries from all deployed specs | All expected entries present |
| Agent SOUL.md completeness | Grep each agent SOUL.md for required keywords | All required content present |
| Profile format compliance | Sample 5 profiles from Archive/, check YAML and detail format | All sampled profiles conform |
| JSON file validity | `jq .` on all JSON files | All parse successfully |
| .bak file existence | Check for index.json.bak and recipe.json.bak | Both exist (if Baker has run) |

### Dynamic Validation (requires running system)

| Test | Method | Pass Condition | Destructive? |
|------|--------|---------------|-------------|
| Baker manual run | `openclaw run breadcrumb-baker` | Completes without error, baker.log updated | No |
| Heartbeat trigger test | Send test message matching recall pattern | CORRECTIONS.md receives entry | Adds test correction (cleanup needed) |
| baker-reindex.sh execution | `baker-reindex.sh tool "test-search"` | Returns valid stdout format | No |
| Lock file creation | Trigger a write agent, check for .lock file | .lock appears and disappears | No |
| Cold start test | Fresh session → task using documented capability | Agent finds capability via recipe.json | No |

**Note:** Dynamic tests marked "cleanup needed" create artifacts that should be removed after the audit. The Auditor will note these in the report.

---

## 9. Common Deployment Failures

Based on observed implementation patterns, these are the most frequently missed items when an agent deploys from spec:

| Rank | Failure | Why Agents Skip It | Impact |
|------|---------|-------------------|--------|
| 1 | **Enhanced tier not wired** | Agent defaults to Core tier path; doesn't check for available infra | Semantic search, notifications, and Vector DB hygiene all missing |
| 2 | **Cron entries not created** | Agent creates files but doesn't modify system crontab | Nothing runs on schedule — entire system is inert |
| 3 | **Heartbeat patterns not registered** | Agent deploys agents but skips heartbeat configuration | No real-time triggers; everything waits for cron |
| 4 | **Standing Directive not added to other agents** | Agent deploys Baker but doesn't update every other agent's SOUL.md | Other agents don't check recipe.json pre-flight |
| 5 | **CONFIG.md parameters incomplete** | Agent creates CONFIG.md with some values; skips new v4.1/v3.1/v2.1 parameters | Locking, cool-down, orphan lifecycle all use defaults or are inert |
| 6 | **baker-reindex.sh not created** | Bridge spec references it; agent leaves it as a future TODO | Recall failure routing for tools is broken |
| 7 | **Sentinel directory not created** | Small detail in Bridge build order; easy to skip | Overrun detection doesn't function |
| 8 | **Validation steps not run** | Agent says "done" after creating files; doesn't execute Build Order verification steps | Problems aren't discovered until the operator notices |
| 9 | **Budget sum exceeds context_budget_tokens** | Agent copies default values without checking arithmetic | Context Assembler silently truncates lower-priority slots |
| 10 | **Lab Manager heartbeat script missing** | Agent deploys weekly cron but skips daily heartbeat | 6-day blind spot for catastrophic failures |

---

## 10. Schedule & Invocation

### Default: On-Demand

The Auditor is primarily an on-demand tool. Run it:

- **After initial deployment** of any spec.
- **After upgrading** a spec to a new version (e.g., v4.0 → v4.1).
- **When something seems wrong** but the Lab Manager report looks fine (implementation gap, not runtime issue).
- **After an agent says "done"** implementing a spec.

### Invocation

```bash
# Manual trigger
openclaw run deployment-auditor

# Or if the operator asks in conversation:
"audit the deployment"
"check if everything was set up correctly"
"verify the implementation"
"what's missing from the setup"
```

### Optional: Scheduled Compliance Check

For environments where configuration drift is a concern (multiple operators, frequent agent changes), the Auditor can be scheduled:

```bash
# Monthly compliance check (optional)
0 12 1 * * openclaw run deployment-auditor
```

This is not recommended for most setups — the Auditor is most valuable when targeted, not routine. The Lab Manager handles ongoing operations monitoring.

---

## 11. Build Order

### Step 1: Create Agent Directory

```bash
mkdir -p ~/agents/deployment-auditor
```

### Step 2: Deploy the SOUL.md

Save the SOUL.md from Section 4 into `agents/deployment-auditor/SOUL.md`.

### Step 3: Place Spec Documents

Copy or symlink all spec markdown files to the auditor's spec directory:

```bash
mkdir -p ~/.openclaw/workspace/specs/
cp /path/to/specs/*.md ~/.openclaw/workspace/specs/
```

### Step 4: Add CONFIG.md Parameter

```yaml
auditor_spec_directory: "~/.openclaw/workspace/specs/"
```

### Step 5: Create the Breadcrumb

If the Breadcrumb System is deployed:

```markdown
---
id: agent_deployment-auditor
name: Deployment Auditor
type: agent
status: active
created: YYYY-MM-DD
updated: YYYY-MM-DD
owner: shared
source: ~/agents/deployment-auditor/
token_estimate: 0
tags: [operations, audit, verification, deployment]
deps: [openclaw, jq]
---

# Purpose
Verify that deployed OpenClaw infrastructure specs have been implemented
completely and correctly. Produces a gap analysis comparing specs against
actual implementation.

# How It Works
1. Detects which specs are deployed and what tier is available
2. Walks each spec's checklist comparing expected vs. actual state
3. Runs validation tests from each spec's Build Order
4. Generates a prioritized audit report with specific fixes

# Usage
## Commands
\```bash
# Manual trigger (primary usage)
openclaw run deployment-auditor

# Optional: monthly scheduled compliance check
# 0 12 1 * * openclaw run deployment-auditor
\```

## Outputs
| Output | Location | Format |
|--------|----------|--------|
| Audit report | ~/lab-reports/YYYY-MM-DD-audit-report.md | Markdown |

# Gotchas
- 📌 Strictly read-only — never modifies any file
- 📌 Run AFTER deployment, not during
- 📌 Dynamic validation tests may create artifacts (noted in report)
- ⚠️ Cannot test Vector DB connectivity if credentials are in env vars
  that the Auditor can't access — will report "UNABLE TO VERIFY"
- 📌 Audit report goes to ~/lab-reports/ alongside Lab Manager reports

# Changelog
- **YYYY-MM-DD**: Created (v1.0)
```

### Step 6: Run Baker

If the Breadcrumb System is deployed, run Baker manually to index the Auditor breadcrumb.

### Step 7: Run the Auditor

Execute the Deployment Auditor for the first time to verify your current implementation:

```bash
openclaw run deployment-auditor
```

Review the audit report at `~/lab-reports/YYYY-MM-DD-audit-report.md`. Fix findings in priority order (🔴 → 🟡 → 🔵). Re-run the Auditor after fixes to confirm.

---

## Summary

| Aspect | Value |
|--------|-------|
| **Agent count** | 1 |
| **Cron jobs** | 0 (on-demand by default) |
| **New file formats** | 0 (markdown report) |
| **New storage layers** | 0 (uses existing ~/lab-reports/) |
| **New extension points** | 1 (auditor_spec_directory) |
| **Impact on other specs** | None (read-only) |
| **Primary use case** | Post-deployment verification |
| **Secondary use case** | Post-upgrade compliance check |
| **Key differentiator from Lab Manager** | Checks implementation completeness, not runtime health |

---

*The Auditor exists because "done" doesn't mean "correct." Run it once after every deployment. If it comes back clean, trust the Lab Manager for everything else.*
