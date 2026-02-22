# OpenClaw CRM System — Relationship Intelligence Spec

*Version: 2.0 | Spec ID: `openclaw-crm-system`*
*Audience: Any OpenClaw operator or agent implementing relationship intelligence*
*License: Share freely. Customize via CONFIG.md.*

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Design Principles](#2-design-principles)
3. [Prerequisites](#3-prerequisites)
4. [Data Schema & File Structure](#4-data-schema--file-structure)
5. [Agent Definition: CRM Miner](#5-agent-definition-crm-miner)
6. [Staleness & Confidence Rules](#6-staleness--confidence-rules)
7. [Heartbeat Triage Patterns](#7-heartbeat-triage-patterns)
8. [Extension Points & Bridge Hooks](#8-extension-points--bridge-hooks)
9. [Usage Workflows](#9-usage-workflows)
10. [Build Order](#10-build-order)

---

## 1. Purpose

Agents interact with the operator about their relationships — family, friends, colleagues, and preferred tools or places. Without a structured way to capture this, details are lost in raw chronological memory. 

This specification defines a standalone **Customer Relationship Management (CRM) system** for OpenClaw. It passively extracts facts about people, places, and things from daily conversations, structuring them into machine-readable, token-efficient profiles. It guarantees the agent knows who people are, what they like, and how they relate to the operator, without needing to be told repeatedly.

---

## 2. Design Principles

1. **Extraction is passive.** The agent listens and captures. It does not proactively interrogate the operator for data.
2. **Confidence decays; truth is timestamped.** Old information is flagged as stale. The agent must know the difference between a verified fact and an outdated assumption.
3. **Operator corrections are absolute.** If the operator corrects a profile detail, that value receives a 1.0 confidence score and overwrites previous data.
4. **Decoupled architecture.** This system works independently. If the Persistent Memory or Breadcrumb systems are present, integration is handled purely through `CONFIG.md` extension points.
5. **One entity, one file.** Profiles are flat markdown files with YAML frontmatter. This guarantees portability, manual editability, and exact-match deduplication.

---

## 3. Prerequisites

### Required

| Component | Purpose | Notes |
|-----------|---------|-------|
| OpenClaw instance | Agent runtime | Any supported platform |
| `Archive/` directory | Profile storage | Flat markdown files |
| PostgreSQL | ChatMessage storage | Or your configured DB for log parsing |
| Cron scheduling | Extraction automation | OpenClaw native |

### Optional (Recommended)

| Component | Purpose | Impact If Absent |
|-----------|---------|-----------------|
| Vector DB | Semantic search across profiles | Falls back to keyword grep |

---

## 4. Data Schema & File Structure

All profiles live in `~/.openclaw/workspace/Archive/` categorized by type (`People/`, `Places/`, `Things/`). 

### File Naming
- **Format:** `{slug}.md` (lowercase, hyphens only). Example: `alex-rivera.md`
- **Collision handling:** If names overlap (e.g., two people named John), append context: `john-smith-work.md`.

### Profile Template
Every profile must conform strictly to this format to allow exact parsing by the Context Assembler and the `crm-miner`.

---
id: person_{slug}
name: {Full Name}
type: person
relationship: {relationship to operator}
created: YYYY-MM-DD
last_verified: YYYY-MM-DD
confidence: {0.0-1.0}
---

# {Full Name}

## Details
- **birthday:** {Value} `[verified: YYYY-MM-DD]` `[confidence: 0.X]`
- **prefers:** {Value} `[verified: YYYY-MM-DD]` `[confidence: 0.X]`

## Notes
[Freeform observations or pointers to external integrations]

## History
- **prefers:** {Old Value} (Updated YYYY-MM-DD)

---

## 5. Agent Definition: CRM Miner

**Name:** `crm-miner`
**Location:** `agents/crm-miner/`
**Schedule:** Cron — daily (default 06:30) + immediate execution on heartbeat triggers.

### SOUL.md

# CRM Miner

## Role
Relationship Intelligence Specialist

## Mission
Build and maintain rich profiles of every person, place, and thing in the
operator's world. Ensure the main agent possesses deep, accurate context 
about the operator's life without requiring repetitive explanations.

## What You Extract
- **People:** names, relationships to operator, roles, preferences, birthdays
- **Places:** restaurants, offices, cities, context about why they matter
- **Things:** products, tools, hobbies, interests, dislikes

## Rules for Profile Generation
1. **One profile per entity.** Never create duplicate profiles. Always search 
   `Archive/People/`, `Archive/Places/`, and `Archive/Things/` before creating new files.
2. **Check exclusion source.** If CONFIG.md defines `crm_entity_exclusion_source`, 
   check that file before creating profiles. Entities matching keys in the 
   exclusion source MUST NOT be profiled. Instead, add a reference to the Notes 
   section of the operator's profile.
3. **Update, don't append.** When new info about an existing entity arrives, 
   update the profile in place. 
4. **Preserve History.** If new info conflicts with existing profile data, 
   update the detail field, log the old value in the `## History` section at 
   the bottom of the profile, and reset `last_verified` to today's date.
5. **Negation handling.** If the operator says "I don't like X", capture it 
   as "hates: X" or "dislikes: X", never "likes: X".
6. **Prioritize.** Process priority-flagged messages from the ChatMessage 
   table first.

## Vector DB Protocol [If Enhanced]
- After writing the markdown file, generate an embedding of the file's contents.
- Tag the embedding with `entity_id: [id]`, `type: [type]`, and `status: active`.
- If updating a profile, overwrite the existing embedding matching that `entity_id`.

## Profile Template Compliance
You must strictly use the YAML frontmatter and markdown structure defined in 
the CRM System Specification. Every detail line MUST end with 
`[verified: YYYY-MM-DD] [confidence: 0.X]`.

---

## 6. Staleness & Confidence Rules

### Confidence Scoring
Every extracted fact requires a confidence score.

| Source | Base Confidence |
|--------|----------------|
| Operator explicit correction | 1.0 |
| Explicit operator statement | 0.95 |
| Inferred from context | 0.60 |
| Third-party mention | 0.50 |

### Staleness Thresholds
The `last_verified` timestamp dictates how the agent utilizes the data. Thresholds are configured in `CONFIG.md` (`staleness_threshold_days`, default 90).

| Age Since Last Verified | Status | Required Agent Behavior |
|------------------------|--------|----------------|
| 0 – 90 days | **Fresh** | Serve confidently as absolute fact. |
| 91 – 180 days | **Stale** | Serve with an internal hedge. Use framing like "Last I recall..." Do not assert as current fact. |
| 181+ days | **Expired** | Do not serve proactively. If directly relevant to the current task, ask: "You mentioned [X] a while back — is that still the case?" |

### Verification Refresh
Any time the operator mentions a tracked entity:
1. The CRM Miner evaluates if the mention confirms, updates, or contradicts the existing profile.
2. Updates `last_verified` to the current date.
3. If confirmed, boosts confidence by `+0.05` (capped at 1.0).

---

## 7. Heartbeat Triage Patterns

To ensure immediate capture of critical relationship data, the CRM system registers these regex patterns with the OpenClaw Heartbeat Triage. Matches flag the message with `priority: true` for the `crm-miner`.

/my (name|birthday|job|title|employer|wife|husband|partner|kid|son|daughter|dog|cat|pet) is/i
/i (work|live|moved|started|quit|left|joined) (at|in|to|from)/i
/[A-Z][a-z]+'s birthday/i
/(favorite|favourite|fav) .{1,30} is/i
/my [a-z]+ is named/i
/i (bought|subscribed to) .*/i

---

## 8. Extension Points & Bridge Hooks

This spec integrates cleanly with other OpenClaw systems without hard dependencies. Configure these in the CRM `CONFIG.md`:

### Breadcrumb Exclusion Hook
**Parameter:** `crm_entity_exclusion_source`
**Purpose:** Prevents the CRM Miner from creating "Thing" profiles for scripts, tools, or APIs that are already documented by the Breadcrumb System.
**Value:** `~/breadcrumb-trail/recipe.json` (if Breadcrumb spec is active).

### Context Assembler Hook
**Parameter:** `crm_context_budget` (default: 1000 tokens)
**Purpose:** Instructs the OpenClaw Context Assembler to allocate token space for CRM profiles at session start. The assembler searches the last 48 hours of conversation logs, matches entity names to `Archive/` filenames, and loads the profiles up to the token limit.

---

## 9. Usage Workflows

### Scenario A: Operator Introduces a New Relationship
1. **Operator:** "Met Dave today. He's a guitarist."
2. **Heartbeat:** Does not hit explicit regex, but is logged to `memory/YYYY-MM-DD.md`.
3. **Cron (06:30):** `crm-miner` reads logs, extracts fact.
4. **Action:** Creates `Archive/People/dave.md` with relationship: "acquaintance", detail: "guitarist", confidence: 0.6 (inferred/casual).

### Scenario B: Operator Corrects a Fact
1. **Operator:** "No, Alex Rivera's birthday is March 15th."
2. **Heartbeat:** Hits regex `/birthday is/i`. Flags priority.
3. **Action:** `crm-miner` runs immediately. Opens `alex-rivera.md`. Moves the old birthday to `## History`. Inserts March 15th with `confidence: 1.0` and today's `last_verified` date.

---

## 10. Build Order

1. **Directories:**
   ```bash
   mkdir -p ~/.openclaw/workspace/Archive/People
   mkdir -p ~/.openclaw/workspace/Archive/Places
   mkdir -p ~/.openclaw/workspace/Archive/Things
   mkdir -p ~/.openclaw/workspace/agents/crm-miner

2.  Deploy Agent: Save the SOUL.md from Section 5 into the crm-miner directory.

3. Configuration: Add crm_entity_exclusion_source and staleness_threshold_days: 90 to your primary CONFIG.md.

4. Heartbeat: Add the regex patterns from Section 7 into your OpenClaw triage configuration.

5. Schedule: Add 30 6 * * * openclaw run crm-miner to your crontab.