# OpenClaw Breadcrumb System — Complete Specification

*Version: 3.6 | Spec ID: `openclaw-breadcrumb-system`*

---

## v3.6 — 2026-02-23

**NEW: Bread Box (renamed index), Registration Hook (`register-crumb.sh`), Baker role refined to reconciliation**

### What Changed

1. **Bread Box** — `bread-box.json` renamed to `bread-box.json`. The Breadcrumb Trail is now the Bread Box — fits the bakery theme and clarifies function ("what bread has been made and where it lives").
2. **Registration Hook** — `register-crumb.sh` provides immediate, atomic registration at build time. Closes the temporal gap between "thing exists" and "system knows thing exists."
3. **Baker Role Refined** — Baker shifts from primary registration mechanism to reconciliation agent (catching drift, stale data, orphans, unregistered components).

---

*Version: 3.5 — Enforcement Language, Remediation Queue, Executable Cold Start Validation, Build Receipts*
*Version: 3.4 — Multi-Directory Support, Gap Notification, Enhanced Governance*
*Version: 3.3 — Governance, Accountability & Verification*
*Version: 3.2 — Gap Detection in Baker*
*Version: 3.1 — Atomic Writes + Orphan Lifecycle*

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Design Principles](#2-design-principles)
3. [Compliance Language](#3-compliance-language)
4. [Prerequisites](#4-prerequisites)
5. [Core Concepts](#5-core-concepts)
6. [Naming Convention](#6-naming-convention)
7. [Breadcrumb Template](#7-breadcrumb-template)
8. [Type-Specific Extensions](#8-type-specific-extensions)
9. [Where Breadcrumbs Live](#9-where-breadcrumbs-live)
10. [The Bread Recipe](#10-the-bread-recipe-capabilities-manifest)
11. [The Bread Box](#11-the-bread-box)
12. [Component Hierarchy Rules](#12-component-hierarchy-rules)
13. [Registration Hook](#13-registration-hook)
14. [The Breadcrumb Baker](#14-the-breadcrumb-baker)
15. [Remediation Queue](#15-remediation-queue)
16. [Executable Cold Start Validation](#16-executable-cold-start-validation)
17. [Build Receipts](#17-build-receipts)
18. [Agent Standing Directive](#18-agent-standing-directive)
19. [Gap Governance](#19-gap-governance)
20. [Usage Workflows](#20-usage-workflows)
21. [Governance](#21-governance)
22. [Build Order](#22-build-order)
23. [Changelog](#23-changelog)

---

## 1. Purpose

### The Problem

Agents build tools, integrations, scripts, and workflows. The next session, they have no memory of what was built, what it does, how to run it, or even that it exists. The operator ends up re-explaining everything, or worse, the agent rebuilds something that already exists.

### The Solution

Every component gets a **breadcrumb** — a structured markdown file that explains exactly what the component is, how it works, and how to use it. All breadcrumbs are cataloged in a **bread box** (a central catalog). A **bread recipe** (capabilities manifest) tells every agent what tools and integrations are available to them before they even start a task. A **registration hook** (`register-crumb.sh`) provides immediate registration at build time. A **Breadcrumb Baker** agent runs periodically as a reconciliation agent to catch drift, validate breadcrumbs, and keep the system healthy.

---

## 2. Design Principles

1. **Breadcrumbs live with their components** (in-situ). The bread box is a catalog, not a copy.
2. **Every agent breadcrumbs what it builds.** The Baker reconciles — it does not retroactively document things it didn't build.
3. **The cold start test:** Validated programmatically via `cold-start-check.sh`. A breadcrumb MUST pass all executable checks (Section 16). *(v3.5: Replaced subjective question with deterministic validation.)*
4. **One source of truth.** No duplicate breadcrumbs. The in-situ file is the canonical version.
5. **Capabilities before tasks.** Every agent checks the bread recipe before starting work to know what tools are available.
6. **Token awareness.** Every breadcrumb carries a token estimate so that any token-budgeted system can plan reads before committing context window space.
7. **Extend, don't couple.** This spec works standalone.
8. **Atomic writes protect data.** All writes to critical bread box and recipe files use an atomic write protocol.
9. **Orphans have a lifecycle.** Auto-drafted orphan breadcrumbs expire after 30 days if not validated.
10. **Governance requires accountability.** Gaps MUST have owners, not just be logged.
11. **Detection ≠ remediation.** The Baker detects problems; remediation tasks are routed to responsible agents via the Remediation Queue (Section 15). *(v3.5: Replaces passive gap logging with active task routing.)*
12. **Enforcement over expectation.** Where a requirement can be validated programmatically, it is classified as MUST and enforced by tooling. Best-effort expectations are classified as SHOULD. *(v3.5: New.)*
13. **Register at build time, reconcile on schedule.** The write path (registration hook) matches the read path (recipe → bread box → crumb). The Baker catches what the hook missed — it is not the primary registration mechanism. *(v3.6: New.)*

---

## 3. Compliance Language

*(v3.5: New section.)*

This specification uses RFC 2119 keywords to distinguish enforceable requirements from best-effort expectations.

| Keyword | Meaning | Enforcement |
|---------|---------|-------------|
| **MUST** | Non-negotiable. System validates programmatically. Non-compliance is a logged failure, generates a remediation task, and affects scorecard metrics. | Baker, `cold-start-check.sh`, `register-crumb.sh`, Remediation Queue |
| **MUST NOT** | Prohibited. Programmatically detected and flagged as failure. | Baker, validation scripts |
| **SHOULD** | Recommended. System cannot reliably validate. Best-effort, acknowledged as behavioral. | No automated enforcement |
| **SHOULD NOT** | Discouraged. Not programmatically enforced. | No automated enforcement |

### Classification Criteria

A directive is classified as **MUST** if and only if:
1. A script, the Baker, or another automated process can detect non-compliance
2. The detection produces a binary pass/fail result
3. The failure can be expressed as a structured remediation task

Everything else is **SHOULD**.

---

## 4. Prerequisites

### Required

| Component | Purpose | Notes |
|-----------|---------|-------|
| OpenClaw instance | Agent runtime | Any supported platform |
| Filesystem access | Breadcrumb storage | Agents need read/write to workspace directories |
| Cron scheduling | Baker automation | OpenClaw native or platform equivalent |
| `jq` | Bread box querying | Standard Linux utility |
| `cold-start-check.sh` | Breadcrumb validation | See Section 16. *(v3.5: New.)* |
| `register-crumb.sh` | Build-time registration | See Section 13. *(v3.6: New.)* |

### Optional (Recommended)

| Component | Purpose | Impact If Absent |
|-----------|---------|-------------------|
| NFS / shared storage mount | Multi-agent access to breadcrumbs | Single-machine setups work fine without it |
| Multiple agents | Distributed builds | System works with a single agent |
| Notification system | Gap alerts | Operators notified of missing capabilities |

---

## 5. Core Concepts

| Concept | What It Is | File |
|---------|-----------|------|
| **Breadcrumb** | A structured doc for a single component | `{type}.{name}.crumb.md` |
| **Bread Box** | Central catalog of all breadcrumbs | `bread-box.json` + `trail.md` *(v3.6: Renamed from index/Breadcrumb Trail.)* |
| **Bread Recipe** | Capabilities manifest — what can agents do | `recipe.json` |
| **Registration Hook** | Build-time script that validates and registers a new crumb | `register-crumb.sh` *(v3.6: New.)* |
| **Breadcrumb Baker** | Reconciliation agent that catches drift, validates, and maintains system health | Runs on schedule *(v3.6: Role clarified.)* |
| **Orphan** | A breadcrumb for a component with no home directory | Lives in `orphans/` |
| **Gap** | Undocumented capability | Lives in `gaps.md` |
| **Remediation Task** | Structured fix-it task routed to a specific agent | Lives in `~/.openclaw/remediation/` *(v3.5: New.)* |
| **Build Receipt** | Intent file marking an in-progress build | `.build-in-progress` in component directory *(v3.5: New.)* |

---

## 6. Naming Convention

### Rules

- **Breadcrumb ID:** `{type}_{component-name}` — underscores between type and name, hyphens within the component name
- **Filename:** `{type}.{component-name}.crumb.md` — dots between segments, `.crumb.md` extension
- **Directory slug:** `{component-name}/` — hyphens only, lowercase

The `.crumb.md` extension is mandatory. It makes breadcrumbs instantly identifiable in any directory listing, `find` command, or grep. It's still valid markdown — any editor renders it fine.

> **MUST:** All breadcrumbs MUST use the `.crumb.md` extension. Baker flags non-compliant filenames as failures.
>
> **MUST:** Breadcrumb IDs MUST follow `{type}_{component-name}` format. Baker validates on every run.

---

## 7. Breadcrumb Template

```markdown
---
id: script_example-tool
name: Example Tool
type: script
status: active
created: YYYY-MM-DD
updated: YYYY-MM-DD
owner: agent-name
source: ~/path/to/component/
token_estimate: 0
tags: [keyword1, keyword2]
deps: [python3, requests]
---

# Purpose                                    ← MUST
One sentence. What this component does. If you can't say it in one sentence,
you don't understand it yet.

# How It Works                               ← MUST
Numbered steps of the actual flow. Not theory — what literally happens when
this component runs.

1. Step one
2. Step two
3. Step three

# Usage                                      ← MUST
## Commands
Real commands. Copy-paste ready. No placeholders unless absolutely necessary.

\```bash
cd ~/path/to/component
python3 example-tool.py
\```

## Inputs
| Input | Value / Location | Notes |
|-------|-----------------|-------|
| API_TOKEN | .secrets/token.txt | Read permission required |

## Outputs
| Output | Location | Format |
|--------|----------|--------|
| results.json | ~/data/results.json | JSON array |

# Gotchas                                   ← MUST
Failure modes, rate limits, quirks, things that have broken before.
- ⚠️ Known issue or warning
- 📌 Important constraint or limitation
- 🔧 Maintenance note or workaround

# Changelog                                 ← MUST
- **YYYY-MM-DD**: Description of change
- **YYYY-MM-DD**: Created
```

### Section Requirements

*(v3.5: Reclassified from "REQUIRED" labels to MUST/SHOULD with validation method.)*

| Section | Level | Validation Method |
|---------|-------|-------------------|
| YAML Frontmatter (all fields) | **MUST** | `cold-start-check.sh` parses frontmatter, fails on missing fields |
| Purpose | **MUST** | `cold-start-check.sh` checks section exists and is non-empty |
| How It Works | **MUST** | `cold-start-check.sh` checks section exists and is non-empty |
| Usage (with Commands subsection) | **MUST** | `cold-start-check.sh` checks section exists and contains at least one code block |
| Gotchas | **MUST** | `cold-start-check.sh` checks section exists and is non-empty |
| Changelog | **MUST** | `cold-start-check.sh` checks section exists and contains at least one dated entry |
| `source` path exists on disk | **MUST** | `cold-start-check.sh` verifies path exists |
| `deps` are installed | **MUST** | `cold-start-check.sh` checks each dep via `which` or `pip show` |
| Commands in Usage are syntactically valid | **SHOULD** | Not reliably validatable without execution |
| Purpose is genuinely one sentence | **SHOULD** | Sentence boundary detection is unreliable |
| How It Works reflects actual runtime behavior | **SHOULD** | Cannot validate without execution |

---

## 8. Type-Specific Extensions

### API Breadcrumbs

```markdown
# Auth
| Field | Value |
|-------|-------|
| Method | OAuth 2.0 / Bearer token / API key |
| Credentials Location | ~/.secrets/service-creds.json |
| Token Refresh | Auto via refresh_token / Manual |

# Endpoints
| Method | Path | Description |
|--------|------|-------------|
| GET | /resource/list | List resources |
| POST | /resource/create | Create resource |
```

> **MUST:** API breadcrumbs MUST include an Auth section with Credentials Location. Baker validates via `cold-start-check.sh`.

### Workflow Breadcrumbs

```markdown
# Steps
| Order | Component | Breadcrumb ID | Input | Output | Depends On |
|-------|-----------|---------------|-------|--------|------------|
| 1 | download.sh | script_download | Source URLs | ~/data/raw/*.mp3 | — |
| 2 | process.py | script_process | *.mp3 files | ~/data/text/*.txt | Step 1 |

# Trigger
Cron: `0 6 * * *` (daily at 6am)
```

> **MUST:** Workflow breadcrumbs MUST include a Steps table and Trigger section.

### Agent Breadcrumbs

```markdown
# Agent Configuration
| Field | Value |
|-------|-------|
| Model | model-name-and-version |
| System Prompt | ~/agents/agent-name/SOUL.md |
| MCP Tools | tool-1, tool-2, tool-3 |
| Host | Platform / VM / Cloud |

# Capabilities
What this agent is authorized to do:
- Read/write files in ~/workspace/
- Execute scripts in ~/scripts/
- NOT authorized to: [explicit denials]
```

> **MUST:** Agent breadcrumbs MUST include Agent Configuration table and Capabilities section.

---

## 9. Where Breadcrumbs Live

### Layer 1: In-Situ (Source of Truth)

Every component's breadcrumb MUST live in its own directory, next to the code:

```
~/agents/example-bot/
├── main.py
├── run.sh
└── agent.example-bot.crumb.md         ← breadcrumb lives here
```

```
~/scripts/morning-brief/
├── generate_brief.sh
├── fetch_calendar.sh
├── deliver_brief.sh
└── script.morning-brief.crumb.md       ← breadcrumb
```

### Layer 2: Central Hub

```
~/breadcrumb-trail/
├── bread-box.json                      ← machine-searchable catalog (v3.6: renamed from bread-box.json)
├── recipe.json                         ← capabilities manifest
├── trail.md                            ← human-readable summary
├── schema.crumb.md                     ← THIS SPEC
├── baker.log                           ← run history
├── gaps/                               ← gap detection
│   ├── gaps.md                         ← gap log
│   └── heartbeat-summary.json          ← for cron integration
└── orphans/                           ← orphaned breadcrumbs
```

> **MUST:** Breadcrumbs MUST NOT be duplicated. The in-situ file is the single source of truth. Baker detects and removes duplicates.

---

## 10. The Bread Recipe (Capabilities Manifest)

### What It Is

The Bread Recipe answers: **"What can I do?"**

Every agent's system prompt MUST include the instruction to check the recipe before starting any task.

### File Location

```
~/breadcrumb-trail/recipe.json
```

### Structure

```json
{
  "version": "1.0",
  "last_updated": "YYYY-MM-DDTHH:MM:SSZ",
  "capabilities": {
    "email": {
      "status": "active",
      "breadcrumb_id": "api_email-access",
      "breadcrumb_path": "orphans/api.email-access.crumb.md",
      "summary": "Read inbox, draft replies, manage labels. No auto-send without approval.",
      "permissions": ["read", "draft", "label"],
      "restrictions": ["no-auto-send", "requires-approval-for-send"]
    },
    "calendar": {
      "status": "active",
      "breadcrumb_id": "api_calendar-access",
      "breadcrumb_path": "orphans/api.calendar-access.crumb.md",
      "summary": "Read/write calendar events.",
      "permissions": ["read", "create", "modify"],
      "restrictions": ["no-delete"]
    }
  }
}
```

### Recipe Sync Enforcement

> **MUST:** Every breadcrumb with `status: active` MUST have a corresponding entry in `recipe.json` capabilities.
>
> **Validation:** The Baker validates this on every run. `register-crumb.sh` enforces this at build time:
> - If breadcrumb is `active` but NOT in recipe.json → Log as COLD START failure + generate remediation task
> - If breadcrumb is in recipe.json but NOT in bread box → Log warning
> - If breadcrumb is `deprecated` but still in recipe.json → Remove from recipe

---

## 11. The Bread Box

*(v3.6: Renamed from "The Breadcrumb Trail (Central Index)" to fit the bakery naming convention.)*

### What It Is

The Bread Box answers: **"What bread has been baked, and where does it live?"**

The recipe tells agents what they *can* do. The bread box tells agents what *has been built* and where to find it.

### bread-box.json

*(v3.6: Renamed from `bread-box.json`.)*

```json
{
  "version": "1.0",
  "last_updated": "YYYY-MM-DDTHH:MM:SSZ",
  "total_breadcrumbs": 0,
  "breadcrumbs": [
    {
      "id": "script_example",
      "name": "Example Script",
      "type": "script",
      "status": "active",
      "path": "~/scripts/example/script.example.crumb.md",
      "source": "~/scripts/example/",
      "token_estimate": 350,
      "tags": ["data", "cron"],
      "owner": "agent-name",
      "created": "YYYY-MM-DD",
      "updated": "YYYY-MM-DD"
    }
  ],
  "tags_index": {
    "data": ["script_example"]
  }
}
```

### Deduplication

The Baker MUST deduplicate before writing bread-box.json:

1. Collect all .crumb.md paths from all scanned directories
2. Sort by modification time (newest first)
3. For duplicate IDs, keep only the newest
4. Log duplicates found and which was kept

### How Agents Use It

```bash
# Search the bread box for a component
jq '.breadcrumbs[] | select(.name | test("keyword"; "i"))' ~/breadcrumb-trail/bread-box.json

# List all active breadcrumbs
jq '.breadcrumbs[] | select(.status == "active") | .name' ~/breadcrumb-trail/bread-box.json

# Get path to a specific crumb
jq -r '.breadcrumbs[] | select(.id == "script_example") | .path' ~/breadcrumb-trail/bread-box.json
```

---

## 12. Component Hierarchy Rules

> **The Independently Callable Rule:**
> If a component can be invoked on its own (by a user, cron job, another script, or another agent), it gets its own breadcrumb.
> If it's only called internally by a parent, it gets documented in the parent's breadcrumb.

---

## 13. Registration Hook

*(v3.6: New section.)*

### The Problem

In v3.5 and earlier, the write path (agent builds → crumb created → Baker eventually discovers and indexes) was decoupled from the read path (agent reads recipe → bread box → crumb). The Baker ran every 6 hours, meaning a newly built component could sit unregistered for hours. The system knew about things only after the Baker crawled for them — a discovery model, not a registration model.

### The Solution

`register-crumb.sh` — a script agents call immediately after creating a `.crumb.md`. It validates the crumb, adds it to the bread box, updates the recipe if applicable, and removes the build receipt. One atomic operation that closes the write path instantly.

The Baker's role shifts from **primary indexer** to **reconciliation agent** — it catches anything the registration hook missed (interrupted sessions, manual file drops, legacy components), validates drift, and cleans up stale data.

### Script Location

```
~/scripts/register-crumb.sh
```

### What It Does

In sequence:

1. **Validate** — Runs `cold-start-check.sh` on the crumb. If any check fails, registration aborts with error output. The crumb is NOT added to the bread box.
2. **Check for duplicates** — Scans `bread-box.json` for existing entry with same `id`. If found, compares modification times and keeps newest.
3. **Add to bread box** — Inserts crumb metadata into `bread-box.json` using Atomic Write Protocol (write to `.tmp`, rename).
4. **Update recipe** — If crumb `status` is `active`, extracts capability entry and adds/updates `recipe.json` using Atomic Write Protocol.
5. **Update trail** — Regenerates `trail.md` (human-readable summary).
6. **Clean up build receipt** — If `.build-in-progress` exists in the component directory, deletes it.
7. **Log** — Appends registration event to `baker.log`.

### Usage

```bash
# Register a single crumb
~/scripts/register-crumb.sh ~/scripts/new-tool/script.new-tool.crumb.md

# Output on success
REGISTERED: script_new-tool
  ✓ cold-start-check passed (17/17)
  ✓ Added to bread-box.json (total: 74)
  ✓ Recipe updated (capability: new-tool)
  ✓ Build receipt removed
  ✓ Logged to baker.log

# Output on failure
REGISTRATION FAILED: script_new-tool
  ✗ cold-start-check failed (15/17)
    ERR_NO_GOTCHAS: Gotchas section missing or empty
    ERR_DEP_MISSING: Dependency 'jq' not found
  → Fix errors and re-run register-crumb.sh
```

### Integration With Agent Workflow

The registration hook replaces the old model of "create crumb and hope Baker finds it." The new build workflow is:

1. Agent creates `.build-in-progress` → **MUST**
2. Agent builds component
3. Agent creates `.crumb.md` → **MUST**
4. Agent runs `register-crumb.sh` → **MUST** *(v3.6: New.)*
5. Registration hook validates, indexes, updates recipe, removes receipt — all atomically
6. Component is immediately discoverable by any agent

If step 4 fails (validation errors), the agent MUST fix the crumb and re-run. The build receipt remains in place, signaling to the Baker that a build is incomplete.

### What the Baker Still Does

With the registration hook handling the write path, the Baker's role narrows to reconciliation:

| Baker Responsibility | Why It's Still Needed |
|---------------------|----------------------|
| Discover unregistered components | Agent may have skipped `register-crumb.sh`, or built outside the standard workflow |
| Validate all crumbs (cold-start-check) | Crumbs may have drifted since registration (source modified, deps removed) |
| Detect stale crumbs | Source files change; crumbs don't auto-update |
| Orphan lifecycle | 30-day expiration still requires periodic checks |
| Deduplication | Edge cases from concurrent builds or manual file operations |
| Recipe sync | Capabilities may have been removed or deprecated since registration |
| Remediation routing | Generates tasks for anything the hook didn't catch |
| Build receipt escalation | Receipts older than 24h without registration |

### Failure Modes

| Scenario | Outcome |
|----------|---------|
| Agent runs `register-crumb.sh` and it passes | Crumb immediately in bread box + recipe. Best case. |
| Agent runs `register-crumb.sh` and it fails | Crumb NOT registered. Agent gets specific errors. Build receipt remains. |
| Agent forgets to run `register-crumb.sh` | Baker catches on next run. Build receipt escalates severity. |
| Agent skips crumb entirely | Baker catches on next run via standard gap detection. |
| `register-crumb.sh` crashes mid-operation | Atomic writes protect bread-box.json and recipe.json. Partial state impossible. Baker reconciles on next run. |

---

## 14. The Breadcrumb Baker

### Overview

The Baker is the **reconciliation agent** for the Breadcrumb System. It catches drift, validates integrity, and routes remediation tasks. It is NOT the primary registration mechanism — that role belongs to `register-crumb.sh` (Section 13). *(v3.6: Role clarified.)*

### Multi-Directory Support (v3.4)

The Baker MUST scan multiple directories. Configure in the Baker's config:

```json
{
  "scan_directories": [
    "~/agents/",
    "~/scripts/",
    "~/tools/",
    "~/skills/",
    "~/workflows/",
    "~/.openclaw/workspace/",
    "~/.openclaw/scripts/"
  ]
}
```

### PHASE 1: DISCOVER (Reconciliation)

*(v3.6: Phase renamed to reflect reconciliation role.)*

- Scan all configured directories for components NOT in `bread-box.json` (unregistered)
- Log new gaps to gaps.md
- Auto-draft only if component has README.md, SKILL.md, or --help
- **MUST:** Check for `.build-in-progress` files. Directories with a build receipt but no `.crumb.md` are flagged as `SEVERITY: HIGH` gaps. *(v3.5)*
- **MUST:** Check for `.crumb.md` files not in `bread-box.json` (registration hook was skipped). Auto-register via `register-crumb.sh`. *(v3.6: New.)*

### PHASE 2: VALIDATE

- Parse YAML frontmatter from all .crumb.md files
- Verify source paths exist
- Calculate token_estimate
- **MUST:** Run `cold-start-check.sh` against every `.crumb.md` file *(v3.5)*
- **MUST:** Recipe sync check — every active crumb MUST be in recipe.json
- Orphan lifecycle checks (30-day expiration)
- **MUST:** Stale detection — flag if source modified since crumb updated

### PHASE 3: DEDUPLICATE + EXTRACT

- Collect crumbs from all directories
- MUST deduplicate by ID (keep newest)
- Extract capabilities from crumbs to recipe.json
- Map: Purpose → summary, tags → permissions, Gotchas → restrictions

### PHASE 4: REBUILD BREAD BOX

*(v3.6: Renamed from "INDEX".)*

- Rebuild bread-box.json using Atomic Write Protocol
- Rebuild trail.md (human-readable)

### PHASE 5: REMEDIATION ROUTING *(v3.5)*

For every failure detected in Phases 1-3:
- **MUST:** Generate a structured remediation task file in `~/.openclaw/remediation/`
- **MUST:** Route task to the responsible agent (determined by `owner` field or gap assignment)
- See Section 15 for task format and routing rules.

### PHASE 6: POST-BAKE VERIFICATION

```
=== BAKER COMPLETE ===
Total breadcrumbs: 73
Active: 64 | Experimental: 9 | Deprecated: 0
Cold start failures: 0            ← MUST be 0 for healthy system
Unregistered crumbs found: 2      ← auto-registered (v3.6)
Open gaps: 55 (↑ 3 new, ↓ 2 resolved)
Unowned gaps >48h: 12             ← MUST trigger WARNING
Duplicate crumbs found: 2 → resolved: 2
Recipe sync errors: 0             ← MUST be 0 for healthy system
Build receipts without crumbs: 0  ← MUST be 0 (v3.5)
Remediation tasks generated: 3
```

**MUST:** If COLD START failures > 0 OR unowned gaps > 10 OR build receipts without crumbs > 0 → Log at WARNING level.

### Schedule

| Run Type | Frequency | What It Does |
|----------|-----------|-------------|
| **Standard reconciliation** | Every 6 hours | Phase 1-6 (full) *(v3.6: Renamed from "Standard scan".)* |
| **Quick rebuild** | On demand | Phase 4 only *(v3.6: Renamed from "Quick reindex".)* |
| **Post-build** | After builds | Validate + rebuild (usually unnecessary if registration hook was used) *(v3.6: Note added.)* |

---

## 15. Remediation Queue

*(v3.5: New section. Replaces passive gap logging as the primary enforcement mechanism.)*

### The Problem With Passive Gap Logging

In v3.4 and earlier, the Baker detected problems and logged them to `gaps.md`. This required the operator or an agent to manually discover, interpret, and act on the log. Nothing routed the problem to the agent responsible. Nothing created a concrete deliverable.

### The Solution

When the Baker detects a failure, it MUST generate a **remediation task** — a structured JSON file routed to the responsible agent's queue. Agents MUST check their remediation queue at session start.

### Task File Location

```
~/.openclaw/remediation/
├── bobby/                          ← per-agent directories
│   ├── REM-001.json
│   └── REM-002.json
├── rob/
│   └── REM-003.json
└── unassigned/                     ← tasks with no identifiable owner
    └── REM-004.json
```

### Task File Format

```json
{
  "id": "REM-001",
  "created": "2026-02-23T12:00:00Z",
  "severity": "HIGH",
  "type": "missing_crumb",
  "component": "~/scripts/new-tool/",
  "owner": "bobby",
  "description": "Component directory has .build-in-progress but no .crumb.md",
  "acceptance_criteria": "Valid .crumb.md exists, passes cold-start-check.sh, and is registered via register-crumb.sh",
  "status": "OPEN",
  "resolved": null
}
```

### Task Types

| Type | Severity | Trigger | Acceptance Criteria |
|------|----------|---------|---------------------|
| `missing_crumb` | HIGH (if build receipt exists) / MEDIUM (otherwise) | Component found with no `.crumb.md` | Valid crumb passes `cold-start-check.sh` and is registered |
| `cold_start_failure` | HIGH | Crumb fails `cold-start-check.sh` | Crumb passes all checks |
| `recipe_sync` | MEDIUM | Active crumb not in recipe.json | Capability entry exists in recipe.json |
| `stale_crumb` | LOW | Source modified after crumb `updated` date | Crumb `updated` date ≥ source modified date |
| `unowned_gap` | MEDIUM | Gap open > 48h with no owner | Owner assigned and acknowledged |
| `stalled_gap` | HIGH | Gap in IN_PROGRESS > 7 days | Gap resolved or ETA updated |
| `duplicate_crumb` | LOW | Multiple crumbs for same component ID | Single canonical crumb remains |
| `unregistered_crumb` | LOW | Crumb exists but not in bread-box.json | Crumb registered via `register-crumb.sh` *(v3.6: New.)* |

### Routing Rules

1. If the crumb has an `owner` field → route to that agent
2. If the component is in a gap with an assigned owner → route to gap owner
3. If no owner can be determined → route to `unassigned/`
4. **MUST:** Unassigned tasks > 48h are flagged in the Baker's post-bake summary as WARNING

### Agent Consumption

Agents MUST check `~/.openclaw/remediation/{agent-name}/` at session start and process OPEN tasks before beginning new work.

When an agent resolves a task:
1. Set `status` to `RESOLVED`
2. Set `resolved` to current timestamp
3. Baker verifies on next run and archives the task

---

## 16. Executable Cold Start Validation

*(v3.5: New section. Replaces the subjective "Could an agent with zero context use this?" question with deterministic checks.)*

### The Problem With Subjective Validation

The v3.4 cold start test asked: *"Could an agent with zero prior context use a component with ONLY the breadcrumb?"* This is a good principle but an unenforceable test. The agent evaluating the question is subject to the same token pressure and behavioral drift as the agent that wrote the crumb.

### The Solution

`cold-start-check.sh` — a deterministic validation script that produces binary pass/fail results per check. No LLM judgment involved.

### Script Location

```
~/scripts/cold-start-check.sh
```

### What It Validates

| Check | Pass Condition | Fail Code |
|-------|----------------|-----------|
| Frontmatter exists | YAML block between `---` markers | `ERR_NO_FRONTMATTER` |
| `id` field present | Non-empty value | `ERR_MISSING_ID` |
| `name` field present | Non-empty value | `ERR_MISSING_NAME` |
| `type` field present | Non-empty value | `ERR_MISSING_TYPE` |
| `status` field present | Value in [active, experimental, broken, deprecated] | `ERR_MISSING_STATUS` |
| `owner` field present | Non-empty value | `ERR_MISSING_OWNER` |
| `source` field present | Non-empty value | `ERR_MISSING_SOURCE` |
| `source` path exists | Directory or file exists on disk | `ERR_SOURCE_NOT_FOUND` |
| `deps` are installed | Each dep resolvable via `which`, `pip show`, or `npm list -g` | `ERR_DEP_MISSING` |
| Purpose section exists | `# Purpose` header followed by non-empty content | `ERR_NO_PURPOSE` |
| How It Works section exists | `# How It Works` header followed by non-empty content | `ERR_NO_HOW_IT_WORKS` |
| Usage section exists | `# Usage` header followed by non-empty content | `ERR_NO_USAGE` |
| Usage contains code block | At least one fenced code block under Usage | `ERR_NO_COMMANDS` |
| Gotchas section exists | `# Gotchas` header followed by non-empty content | `ERR_NO_GOTCHAS` |
| Changelog section exists | `# Changelog` header with at least one dated entry | `ERR_NO_CHANGELOG` |
| Naming convention | Filename matches `{type}.{name}.crumb.md` pattern | `ERR_BAD_FILENAME` |
| ID matches filename | `id` field matches filename-derived ID | `ERR_ID_MISMATCH` |

### Usage

```bash
# Validate a single crumb
~/scripts/cold-start-check.sh ~/agents/bobby/agent.bobby.crumb.md

# Output on success
PASS: agent.bobby.crumb.md (17/17 checks passed)

# Output on failure
FAIL: agent.bobby.crumb.md (15/17 checks passed)
  ERR_NO_GOTCHAS: Gotchas section missing or empty
  ERR_DEP_MISSING: Dependency 'jq' not found
```

### Integration

- **`register-crumb.sh`** runs `cold-start-check.sh` as its first step. Registration fails if validation fails. *(v3.6: New integration point.)*
- **Baker** runs `cold-start-check.sh` against every `.crumb.md` during Phase 2 (VALIDATE). Any failure generates a remediation task of type `cold_start_failure`.

---

## 17. Build Receipts

*(v3.5: New section.)*

### The Problem

An agent can build a component, get interrupted or complete the session, and leave no breadcrumb. The Baker discovers the gap hours later on its next scheduled run. There is no way to distinguish "old component that predates the breadcrumb system" from "agent skipped documentation on something it just built."

### The Solution

When an agent begins building a new component, it MUST create a `.build-in-progress` file in the component's directory. This file signals intent and creates accountability.

### File Format

```
~/scripts/new-tool/.build-in-progress
```

```json
{
  "agent": "bobby",
  "started": "2026-02-23T14:00:00Z",
  "task": "Building new-tool script for API monitoring",
  "expected_crumb": "script.new-tool.crumb.md"
}
```

### Lifecycle

1. Agent creates `.build-in-progress` at build start → **MUST**
2. Agent builds component
3. Agent creates `.crumb.md` → **MUST**
4. Agent runs `register-crumb.sh` → **MUST** *(v3.6: Registration hook removes receipt automatically.)*

If the agent runs `register-crumb.sh` successfully, the receipt is deleted as part of the registration process. If the agent skips registration or registration fails:
- Build receipt remains in place
- Baker catches on next run
- Gap is logged with `SEVERITY: HIGH` (this was a known build, not an undiscovered legacy component)
- Remediation task generated and routed to the agent named in the receipt
- Build receipt age is tracked — receipts older than 24h without registration are escalated

### Baker Detection

| Condition | Severity | Action |
|-----------|----------|--------|
| `.build-in-progress` + no `.crumb.md` | HIGH | Remediation task → agent |
| `.build-in-progress` + valid `.crumb.md` (not registered) | MEDIUM | Auto-register via `register-crumb.sh`, log warning *(v3.6: Baker now auto-registers.)* |
| `.build-in-progress` + registered `.crumb.md` | INFO | Log warning: receipt not cleaned up |
| `.build-in-progress` older than 24h | CRITICAL | Escalate in post-bake summary |

---

## 18. Agent Standing Directive

Every agent MUST include these rules in its system prompt.

*(v3.5: Each rule classified as MUST or SHOULD with validation method noted.)*

### RULE 1: PRE-FLIGHT CHECK

**Classification: SHOULD** *(Cannot be enforced without orchestrator-level gating. Behavioral expectation.)*

Before executing any task involving external services, APIs, tools, or any component not built in this session:

1. Read ~/breadcrumb-trail/recipe.json
2. Identify which capabilities are needed
3. Read corresponding .crumb.md files
4. THEN proceed

### RULE 2: SEARCH BEFORE YOU ASK

**Classification: SHOULD** *(Cannot be programmatically validated. Behavioral expectation.)*

When unsure how something works:

1. Search bread-box.json: `jq '.breadcrumbs[] | select(.name | test("keyword"; "i"))' ~/breadcrumb-trail/bread-box.json`
2. Read matching .crumb.md
3. ONLY ask if breadcrumb missing or says "UNKNOWN"

### RULE 3: WHEN YOU BUILD SOMETHING NEW

**Classification: MUST** *(Baker validates: component directory without `.crumb.md` = failure. Build receipts enforce accountability. Registration hook enforces immediate indexing.)*

- MUST create `.build-in-progress` in component's directory at build start *(v3.5)*
- MUST create `.crumb.md` in component's directory upon completion
- MUST follow naming convention
- MUST run `register-crumb.sh` to validate and register the crumb *(v3.6: New. Replaces manual recipe.json update.)*
- Set token_estimate to 0 (Baker calculates)

### RULE 4: WHEN YOU MODIFY SOMETHING

**Classification: MUST** *(Baker detects stale crumbs where source modified date > crumb updated date.)*

- MUST update breadcrumb's `updated` date
- MUST add changelog entry
- SHOULD re-run `register-crumb.sh` to update bread box and recipe *(v3.6: New. Behavioral because modifications to existing crumbs are harder to gate.)*

### RULE 5: WHEN SOMETHING BREAKS

**Classification: Partial**

- MUST add failure to Gotchas section *(Validated by cold-start-check.sh — Gotchas section must be non-empty.)*
- SHOULD set status to `broken` if can't fix *(Status changes are behavioral; Baker can detect `broken` status but cannot validate whether status should have been changed.)*

### RULE 6: COLD START VALIDATION

**Classification: MUST** *(Enforced by `register-crumb.sh` — registration fails if cold-start-check fails.)* 

Before finalizing any breadcrumb, the crumb MUST pass `cold-start-check.sh`. In practice, this is enforced by `register-crumb.sh` which runs the check automatically:

```bash
~/scripts/register-crumb.sh path/to/component.crumb.md
```

If any check fails, registration aborts. Fix and re-run.

### RULE 7: GAP OWNERSHIP

**Classification: Partial**

- SHOULD acknowledge within 48 hours *(Behavioral — Baker can detect unacknowledged gaps but cannot force acknowledgment.)*
- SHOULD provide ETA or reason for delay *(Behavioral.)*
- MUST update status when resolved *(Baker validates: resolved gaps without updated status are flagged.)*

### RULE 8: REMEDIATION QUEUE *(v3.5)*

**Classification: MUST** *(Baker validates: OPEN remediation tasks in an agent's queue older than 48h trigger WARNING.)*

At session start:
- MUST check `~/.openclaw/remediation/{your-agent-name}/` for OPEN tasks
- MUST process OPEN remediation tasks before beginning new work
- MUST set task status to `RESOLVED` and add timestamp when complete

---

## 19. Gap Governance

### Gap Log Format

```
~/.openclaw/gaps/gaps.md
```

```markdown
| ID | Component | Status | Detected | Owner | ETA | Resolved |
|----|-----------|--------|----------|-------|-----|----------|
| GAP-001 | script/new-tool.sh | OPEN | 2026-02-23 | rob | 2026-02-25 | - |
| GAP-002 | tool.weave2 | IN_PROGRESS | 2026-02-22 | bobby | - | - |
| GAP-003 | api.slack | RESOLVED | 2026-02-20 | - | - | 2026-02-22 |
```

### Gap States

| State | Meaning | Action |
|-------|---------|--------|
| OPEN | New, unassigned | MUST assign owner within 48h |
| IN_PROGRESS | Being worked | MUST update ETA |
| RESOLVED | Fixed | MUST mark resolved date |
| STALLED | No progress > 7 days | MUST escalate → remediation task generated |

### Gap → Remediation Integration *(v3.5)*

Gaps are still logged in `gaps.md` for historical tracking. However, actionable gaps now ALSO generate remediation tasks:

- OPEN gap > 48h with no owner → remediation task type `unowned_gap` → routed to `unassigned/`
- IN_PROGRESS gap > 7 days → remediation task type `stalled_gap` → routed to assigned owner
- Gap severity escalates if a build receipt exists for the component

### Gap Notification

**v3.4 Enhancement:** Gaps can trigger notifications to operators:

1. **Heartbeat integration:** Baker writes summary to `~/.openclaw/gaps/heartbeat-summary.json`
2. **Notification channel:** System reads summary and includes in operator briefings
3. **Escalation:** Unowned gaps > 48h flagged in Baker summary as warning

### Gap Detection Patterns

The system scans for phrases like:
- "can you send..."
- "can you post..."
- "why don't you have..."
- "i need to..."

---

## 20. Usage Workflows

### Workflow A: Agent Needs Tool

1. Agent reads recipe.json → finds capability
2. Agent searches bread-box.json → finds crumb location
3. Agent reads .crumb.md → learns usage
4. Agent executes task

### Workflow B: Agent Builds Something New *(v3.6: Updated to include registration hook.)*

1. Agent creates `.build-in-progress` → **MUST**
2. Agent builds component
3. Agent creates `.crumb.md` → **MUST**
4. Agent runs `register-crumb.sh` → **MUST**
5. Hook validates (cold-start-check), adds to bread box, updates recipe, removes receipt
6. Component immediately discoverable by any agent

### Workflow C: Gap Discovered (Baker Reconciliation)

1. Baker runs → finds unregistered component (not in bread-box.json)
2. Gap logged to gaps.md with timestamp
3. If crumb exists but wasn't registered → Baker auto-registers via `register-crumb.sh` *(v3.6: New.)*
4. If no crumb exists → Remediation task generated and routed to owner
5. Owner creates breadcrumb and runs `register-crumb.sh`
6. Gap marked RESOLVED, remediation task marked RESOLVED

### Workflow D: Cold Start Failure

1. Baker runs → `cold-start-check.sh` fails on a breadcrumb
2. Cold start failure logged
3. Remediation task generated with specific failure codes
4. Builder fixes crumb, re-runs `register-crumb.sh` (which re-validates)
5. Baker re-runs → validates

### Workflow E: Build Receipt Without Crumb *(v3.5)*

1. Baker runs → finds `.build-in-progress` with no `.crumb.md`
2. HIGH severity gap logged
3. Remediation task generated → routed to agent named in receipt
4. Agent creates crumb, runs `register-crumb.sh` (validates, registers, removes receipt)
5. Baker re-runs → validates

---

## 21. Governance

### Status Lifecycle

```
experimental → active → deprecated → archived
     ↓            ↓         ↓
   broken     deprecated   ↓
                            archived
```

| Status | Meaning | Action |
|--------|---------|--------|
| `experimental` | New, untested | Use with caution |
| `active` | Stable, tested | Use freely |
| `broken` | Known failure | MUST NOT use |
| `deprecated` | Replaced/removed | MUST NOT use |

### Quality Reviews

- Baker MUST flag stale/cold-start failures every run
- Baker MUST generate remediation tasks for all failures
- Baker MUST auto-register crumbs found outside bread-box.json *(v3.6: New.)*
- Operator reviews gap summary periodically
- Any agent can update breadcrumb it uses

---

## 22. Build Order

### Step 1: Create Directory Structure

```bash
mkdir -p ~/breadcrumb-trail/orphans
mkdir -p ~/breadcrumb-trail/archive
mkdir -p ~/.openclaw/gaps
mkdir -p ~/.openclaw/remediation/unassigned
```

### Step 2: Install This Spec

Save as: `~/breadcrumb-trail/schema.crumb.md`

### Step 3: Create Initial Files

```bash
# recipe.json
echo '{"version": "1.0", "last_updated": "", "capabilities": {}}' > ~/breadcrumb-trail/recipe.json

# bread-box.json (v3.6: renamed from bread-box.json)
echo '{"version": "1.0", "last_updated": "", "total_breadcrumbs": 0, "breadcrumbs": [], "tags_index": {}}' > ~/breadcrumb-trail/bread-box.json

# gaps.md
echo "# Gap Feedback Log\n\n| ID | Component | Status | Detected | Owner | ETA | Resolved |\n|----|-----------|--------|----------|-------|-----|----------|" > ~/.openclaw/gaps/gaps.md
```

### Step 4: Deploy Scripts *(v3.6: Updated to include register-crumb.sh.)*

```bash
chmod +x ~/scripts/cold-start-check.sh
chmod +x ~/scripts/register-crumb.sh

# Verify cold-start-check
~/scripts/cold-start-check.sh --self-test

# Verify register-crumb (dry run)
~/scripts/register-crumb.sh --dry-run ~/path/to/any.crumb.md
```

### Step 5: Create Breadcrumbs for Existing Components

Walk through every known component and create its `.crumb.md` file in-situ. Register each via `register-crumb.sh` (which validates and indexes in one step).

### Step 6: Deploy the Baker Agent

Deploy the Breadcrumb Baker with its SOUL.md. Execute manually first to validate. Verify Baker runs reconciliation (Phase 1) and can auto-register unregistered crumbs.

### Step 7: Create Per-Agent Remediation Directories

```bash
# Create a directory for each known agent
mkdir -p ~/.openclaw/remediation/bobby
mkdir -p ~/.openclaw/remediation/rob
mkdir -p ~/.openclaw/remediation/unassigned
```

### Step 8: Add Standing Directive to All Agents

Include the Agent Standing Directive (Section 18) in every agent's system prompt. Ensure Rules 3 (build + register) and 8 (Remediation Queue) are included.

### Step 9: Schedule the Baker

```
0 */6 * * * ~/scripts/breadcrumb-baker.sh
```

### Step 10: Verify Registration Hook

```bash
# Create a test crumb and register it
~/scripts/register-crumb.sh ~/test/script.test-component.crumb.md

# Verify it appears in bread box
jq '.breadcrumbs[] | select(.id == "script_test-component")' ~/breadcrumb-trail/bread-box.json
```

### Step 11: Verify Cold Start Validation

```bash
# Run against all crumbs
find ~/agents ~/scripts ~/tools ~/skills ~/workflows -name "*.crumb.md" -exec ~/scripts/cold-start-check.sh {} \;
```

### Step 12: Verify Atomic Write & Backup

```bash
ls -la ~/breadcrumb-trail/bread-box.json.bak
ls -la ~/breadcrumb-trail/recipe.json.bak
```

### Step 13: Verify Gap Detection

```bash
cat ~/.openclaw/gaps/gaps.md
```

### Step 14: Verify Heartbeat Integration

```bash
cat ~/.openclaw/gaps/heartbeat-summary.json
```

### Step 15: Verify Remediation Queue

```bash
# Check for any generated tasks
find ~/.openclaw/remediation/ -name "*.json" -exec cat {} \;
```

### Step 16: Migrate from bread-box.json *(v3.6: One-time migration step.)*

If upgrading from v3.5 or earlier:

```bash
# Rename bread-box.json to bread-box.json
mv ~/breadcrumb-trail/bread-box.json ~/breadcrumb-trail/bread-box.json
mv ~/breadcrumb-trail/bread-box.json.bak ~/breadcrumb-trail/bread-box.json.bak 2>/dev/null

# Update any scripts or agent prompts that reference bread-box.json
grep -r "bread-box.json" ~/agents/ ~/scripts/ ~/breadcrumb-trail/
# Replace all occurrences with bread-box.json
```

---

## 23. Changelog

### v3.6 — 2026-02-23

1. **Bread Box (Renamed from Index/Breadcrumb Trail)**
   - `bread-box.json` renamed to `bread-box.json`
   - "Breadcrumb Trail" concept renamed to "Bread Box" — fits bakery theme
   - Answers: "What bread has been baked, and where does it live?"
   - All references throughout spec updated
   - Migration step added to Build Order (Step 16)

2. **Registration Hook (New Section 13)**
   - `register-crumb.sh` provides immediate, atomic registration at build time
   - Validates crumb (via `cold-start-check.sh`), adds to bread-box.json, updates recipe.json, removes build receipt — all in one atomic operation
   - Closes temporal gap between "thing exists" and "system knows thing exists"
   - Write path now matches read path: build → register → immediately discoverable
   - Agents MUST run `register-crumb.sh` after creating a crumb (Standing Directive Rule 3)

3. **Baker Role Refined**
   - Baker shifts from primary indexer to reconciliation agent
   - Phase 1 renamed to "DISCOVER (Reconciliation)"
   - Phase 4 renamed to "REBUILD BREAD BOX"
   - Baker now auto-registers crumbs found outside bread-box.json
   - Schedule renamed: "Standard scan" → "Standard reconciliation"
   - Baker remains essential for: drift detection, stale crumbs, orphan lifecycle, deduplication, remediation routing

4. **Design Principle 13: Register at Build Time, Reconcile on Schedule**
   - New principle codifying the dual-path architecture
   - Registration hook is the write path; Baker is the safety net

5. **New Remediation Task Type: `unregistered_crumb`**
   - For crumbs that exist on disk but aren't in bread-box.json
   - LOW severity (Baker auto-registers, but logs the miss)

### v3.5 — 2026-02-23

1. **RFC 2119 Enforcement Language**
   - New Section 3: Compliance Language
   - All directives reclassified as MUST (programmatically validatable) or SHOULD (best-effort)
   - Standing Directive rules individually classified with validation method
   - MUST directives tied to specific enforcement mechanisms (Baker, cold-start-check.sh, Remediation Queue)

2. **Remediation Queue (New Section 15)**
   - Baker-detected failures generate structured JSON task files
   - Tasks routed to per-agent directories in `~/.openclaw/remediation/`
   - 7 task types with severity levels and acceptance criteria
   - Agents MUST check queue at session start and process OPEN tasks
   - Unassigned tasks > 48h flagged as WARNING
   - Replaces passive gap logging as primary enforcement mechanism

3. **Executable Cold Start Validation (New Section 16)**
   - `cold-start-check.sh` replaces subjective "could an agent use this?" question
   - 17 deterministic checks with binary pass/fail results
   - Validates frontmatter, required sections, source paths, dependencies, naming
   - Integrated into Baker Phase 2 — runs against every crumb on every bake
   - Failures generate remediation tasks with specific error codes

4. **Build Receipts (New Section 17)**
   - `.build-in-progress` intent files track active builds
   - Agents MUST create receipt at build start, delete when crumb finalized
   - Baker flags receipt-without-crumb as HIGH severity (vs MEDIUM for legacy gaps)
   - Receipts older than 24h escalated to CRITICAL
   - Distinguishes "agent skipped docs" from "old undocumented component"

5. **Baker Phase 5: Remediation Routing**
   - New phase between index and post-bake verification
   - All failures from Phases 1-3 converted to structured remediation tasks
   - Tasks routed by owner field or gap assignment

6. **Design Principle 12: Enforcement Over Expectation**
   - New principle codifying MUST/SHOULD classification criteria

### v3.4 — 2026-02-23

1. **Multi-Directory Support**
   - Baker scans multiple workspace paths (agents/, scripts/, tools/, skills/, workflows/, .openclaw/)
   - Configurable scan_directories in Baker config

2. **Gap Notification (from v3.2)**
   - Gap summary written to heartbeat-summary.json
   - Integration with operator notification channels

3. **Recipe Sync Enforcement (from v3.3)**
   - Baker validates every active breadcrumb has recipe.json entry
   - Logs COLD START failure if missing

4. **Gap Ownership (from v3.3)**
   - Gap log tracks: ID, Component, Status, Detected, Owner, ETA, Resolved
   - States: OPEN → IN_PROGRESS → RESOLVED, plus STALLED escalation
   - Unowned gaps >48h flagged in Baker summary

5. **Post-Bake Verification (from v3.3)**
   - Summary log with actionable metrics
   - Cold start failures, open gaps, unowned gaps, duplicates, recipe sync errors

6. **Stale Breadcrumb Handling (from v3.3)**
   - If source modified since breadcrumb `updated` → log as STALE
   - Baker does NOT auto-fix (detection ≠ remediation)

7. **Duplicate Crumb Detection (from v3.3)**
   - Baker scans for duplicate .crumb.md for same component
   - Keeps in-situ, removes orphans, logs warning

### v3.3 — 2026-02-23

- Governance, accountability, verification added

### v3.2 — 2026-02-22

- Gap Detection Phase added

### v3.1 — 2026-02-22

- Atomic Write Protocol
- Orphan Lifecycle (30-day expiration)

### v3.0 — 2026-02-22

- Initial release

---

*This specification lives at `~/breadcrumb-trail/schema.crumb.md` and is the canonical reference.*
