# OpenClaw CRM System — Relationship Intelligence Spec

*Version: 2.1 | Spec ID: `openclaw-crm-system`*
*Audience: Any OpenClaw operator or agent implementing relationship intelligence*
*License: Share freely. Customize via CONFIG.md.*

---

## Changelog

### v2.1 — 2026-02-22
- **Entity Disambiguation Protocol (§4.1)** — Formal procedure for resolving ambiguous entity references against existing profiles. Provisional profiles created with `confidence: low` when identity cannot be determined.
- **Entity Merge Workflow (§4.2)** — Structured process for merging confirmed-duplicate profiles, preserving all unique data and creating alias redirects.
- **Aliases field** — New `aliases` YAML frontmatter field on all profile types. Supports alternative names, handles, and nicknames for disambiguation matching.
- **Disambiguation section** — New `## Disambiguation` section in profile template for tracking provisional status and merge history.
- **7 additional Heartbeat Triage Patterns** — Location mentions, workplace changes, hobby/interest mentions, health updates, travel plans, milestone events, preference changes (totaling 13 patterns).
- **CONFIG.md addition** — `max_provisional_age_days` (default: 30) for flagging stale provisional profiles.
- **CRM Miner SOUL.md updates** — Added Entity Disambiguation Protocol, Entity Merge Workflow, file locking (`Acquire lock`), and aliases handling directives.

### v2.0 — Initial Release
- Core CRM system: passive extraction, profile schema, confidence scoring, staleness rules.
- CRM Miner agent definition with SOUL.md.
- Heartbeat triage patterns (6 patterns).
- Extension points for Breadcrumb and Context Assembler integration.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Design Principles](#2-design-principles)
3. [Prerequisites](#3-prerequisites)
4. [Data Schema & File Structure](#4-data-schema--file-structure)
   - 4.1 [Entity Disambiguation Protocol](#41-entity-disambiguation-protocol)
   - 4.2 [Entity Merge Workflow](#42-entity-merge-workflow)
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
6. **Disambiguation before creation.** *(v2.1)* Before creating any new profile, the CRM Miner must execute the Entity Disambiguation Protocol (§4.1) to prevent duplicates and resolve ambiguous references.
7. **Aliases are first-class.** *(v2.1)* Every entity may be known by multiple names or handles. The `aliases` field ensures all references resolve to the correct canonical profile.

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

```yaml
---
id: person_{slug}
name: {Full Name}
type: person
relationship: {relationship to operator}
aliases: [{Alias1}, {Alias2}, {@handle}]
created: YYYY-MM-DD
last_verified: YYYY-MM-DD
confidence: {0.0-1.0}
---
```

```markdown
# {Full Name}

## Details
- **birthday:** {Value} `[verified: YYYY-MM-DD]` `[confidence: 0.X]`
- **prefers:** {Value} `[verified: YYYY-MM-DD]` `[confidence: 0.X]`

## Disambiguation
- **status:** {canonical | provisional | merged}
- **provisional_reason:** {Why this profile is provisional, if applicable}
- **provisional_created:** {YYYY-MM-DD, if provisional}
- **merged_from:** [{list of profile IDs merged into this one}]
- **merged_into:** {profile ID this was merged into, if status is "merged"}

## Notes
[Freeform observations or pointers to external integrations]

## History
- **prefers:** {Old Value} (Updated YYYY-MM-DD)
```

### Aliases Field *(v2.1)*

The `aliases` field in YAML frontmatter is a list of alternative names, nicknames, handles, or abbreviations by which the entity is known. This field is critical for the disambiguation protocol.

**Rules:**
- Every known alternative reference to the entity MUST be listed.
- Handles should include the platform prefix (e.g., `@janedoe_42`).
- Nicknames and shortened names should be included (e.g., `Johnny`, `JM`).
- The `name` field remains the canonical/primary name; aliases are secondary references.
- Aliases are case-insensitive for matching purposes.

**Example:**
```yaml
aliases: [Alex, AR, @arivera, Rivera]
```

### Profile Example (Complete)

```yaml
---
id: person_alex-rivera
name: Alex Rivera
type: person
relationship: colleague
aliases: [Alex, AR, @arivera, Rivera]
created: 2026-01-15
last_verified: 2026-02-20
confidence: 0.85
---
```

```markdown
# Alex Rivera

## Details
- **birthday:** March 15th `[verified: 2026-02-20]` `[confidence: 1.0]`
- **prefers:** dark roast coffee `[verified: 2026-01-20]` `[confidence: 0.6]`
- **role:** Senior Engineer at Acme Corp `[verified: 2026-02-10]` `[confidence: 0.95]`

## Disambiguation
- **status:** canonical
- **merged_from:** [person_a-rivera-temp]

## Notes
Met through the operator's work. Plays guitar on weekends.

## History
- **birthday:** March 14th (Updated 2026-02-20)
```

---

### 4.1 Entity Disambiguation Protocol *(v2.1)*

When the CRM Miner encounters a name reference in conversation, it MUST execute this protocol before creating or updating any profile.

#### Step 1: Exact Match
Search all existing profiles by `name` field AND all entries in `aliases` fields (case-insensitive).
- **Match found (single):** Update that profile. Done.
- **Match found (multiple):** Proceed to Step 2.
- **No match:** Proceed to Step 2.

#### Step 2: Contextual Disambiguation
Gather all available context clues from the conversation:
- Relationship descriptors ("my coworker John", "John from the gym")
- Associated entities (mentioned alongside known people/places)
- Activities or roles mentioned
- Temporal context (when was this person last mentioned?)

Compare these clues against existing profiles:
- **Strong match** (≥2 context clues align with one existing profile): Update that profile. Done.
- **Weak match** (1 context clue aligns, or clues are ambiguous): Proceed to Step 3.
- **No match** (new entity entirely): Create a new canonical profile. Done.

#### Step 3: Provisional Profile Creation
If the identity cannot be confidently resolved:

1. Create a new profile with:
   - `confidence: 0.3` (low)
   - `## Disambiguation` section with `status: provisional`
   - `provisional_reason:` describing why disambiguation failed
   - `provisional_created:` set to today's date
2. **Never merge automatically.** Provisional profiles remain separate until the operator confirms identity or sufficient evidence accumulates (≥3 independent context clues converging on a single existing profile).
3. File naming for provisional profiles: append `-provisional` or a disambiguating context slug (e.g., `john-gym.md`, `john-work-provisional.md`).

#### Automatic Promotion
A provisional profile is promoted to `status: canonical` when:
- The operator explicitly confirms the entity's identity.
- Three or more independent conversation references provide consistent, non-contradictory context clues that uniquely identify the entity.
- Confidence score reaches ≥ 0.7 through accumulated evidence.

#### Provisional Profile Expiry
Provisional profiles older than `max_provisional_age_days` (configured in CONFIG.md, default: 30) are flagged for operator review. During the next heartbeat or cron run, the CRM Miner should:
1. List all expired provisional profiles.
2. Present them to the operator for confirmation, merge, or deletion.
3. If no operator action is taken within one additional cycle, mark them with a `⚠️ REVIEW NEEDED` note in the `## Notes` section.

---

### 4.2 Entity Merge Workflow *(v2.1)*

When two profiles are confirmed to represent the same entity (by operator correction or sufficient evidence), the CRM Miner executes the following merge procedure.

#### Merge Rules

1. **Surviving profile:** The older profile (by `created` date) OR the more complete profile (more detail entries) survives. If equal, prefer the one with the higher confidence score.
2. **Acquire lock** on both profile files before beginning the merge to prevent concurrent writes.
3. **Data preservation:** Every unique detail from the retired profile MUST be copied into the surviving profile. If both profiles contain the same field with different values, keep the value with the higher confidence score and move the other to `## History`.
4. **Aliases consolidation:** Merge all `aliases` from both profiles into the surviving profile's `aliases` list. Add the retired profile's `name` to the surviving profile's `aliases` if not already present.
5. **Confidence update:** Set the surviving profile's frontmatter `confidence` to the higher of the two original values.
6. **Update `last_verified`** to today's date.
7. **Update `## Disambiguation`:**
   - Surviving profile: Add the retired profile's `id` to `merged_from` list.
   - Surviving profile: Ensure `status: canonical`.
8. **Retire the merged profile:**
   - Set the retired profile's `confidence` to `0.0`.
   - Set `status: merged` in its `## Disambiguation` section.
   - Set `merged_into:` to the surviving profile's `id`.
   - Clear all `## Details` (they now live in the surviving profile).
   - Add a note: `This profile has been merged into [{surviving name}]({surviving filename})`.
9. **Release locks** on both files.
10. **Vector DB update** (if active): Delete the retired profile's embedding. Re-generate the surviving profile's embedding.

#### Merge Example

**Before merge:** `john-gym.md` (provisional, created 2026-02-01) and `john-smith.md` (canonical, created 2026-01-10).

Operator says: "Oh, John from the gym is John Smith."

**Result:**
- `john-smith.md` survives. Gains gym-related details. `aliases` gains `John from the gym`. `merged_from: [person_john-gym]`.
- `john-gym.md` becomes a stub: `status: merged`, `merged_into: person_john-smith`, details cleared.

---

## 5. Agent Definition: CRM Miner

**Name:** `crm-miner`
**Location:** `agents/crm-miner/`
**Schedule:** Cron — daily (default 06:30) + immediate execution on heartbeat triggers.

### SOUL.md

```markdown
# CRM Miner

## Role
Relationship Intelligence Specialist

## Mission
Build and maintain rich profiles of every person, place, and thing in the
operator's world. Ensure the main agent possesses deep, accurate context
about the operator's life without requiring repetitive explanations.

## What You Extract
- **People:** names, relationships to operator, roles, preferences, birthdays,
  aliases (nicknames, handles, alternative names)
- **Places:** restaurants, offices, cities, context about why they matter
- **Things:** products, tools, hobbies, interests, dislikes

## Rules for Profile Generation
1. **One profile per entity.** Never create duplicate profiles. Always search
   `Archive/People/`, `Archive/Places/`, and `Archive/Things/` before creating
   new files. Search both `name` and `aliases` fields.
2. **Entity Disambiguation Protocol.** Before creating any new profile, execute
   the full Entity Disambiguation Protocol (§4.1 of the CRM System Spec):
   - Step 1: Exact match against names and aliases (case-insensitive).
   - Step 2: Contextual disambiguation using relationship descriptors,
     associated entities, activities, and temporal context.
   - Step 3: If identity cannot be resolved, create a provisional profile
     with `confidence: 0.3` and `status: provisional`. Never merge
     automatically without sufficient evidence.
3. **Entity Merge Workflow.** When two profiles are confirmed to be the same
   entity (by operator correction or ≥3 converging context clues), execute the
   Entity Merge Workflow (§4.2 of the CRM System Spec):
   - Merge into the older/more complete profile.
   - Preserve all unique data from both profiles.
   - Consolidate aliases from both profiles.
   - Create an alias/redirect stub from the retired profile.
4. **Acquire lock before writes.** Before writing to any profile file, acquire
   a file lock to prevent concurrent modification. Release the lock after the
   write is complete. For merges, acquire locks on both files before beginning.
5. **Check exclusion source.** If CONFIG.md defines `crm_entity_exclusion_source`,
   check that file before creating profiles. Entities matching keys in the
   exclusion source MUST NOT be profiled. Instead, add a reference to the Notes
   section of the operator's profile.
6. **Update, don't append.** When new info about an existing entity arrives,
   update the profile in place.
7. **Preserve History.** If new info conflicts with existing profile data,
   update the detail field, log the old value in the `## History` section at
   the bottom of the profile, and reset `last_verified` to today's date.
8. **Negation handling.** If the operator says "I don't like X", capture it
   as "hates: X" or "dislikes: X", never "likes: X".
9. **Prioritize.** Process priority-flagged messages from the ChatMessage
   table first.
10. **Maintain aliases.** When new nicknames, handles, or alternative names
    are discovered for an entity, add them to the `aliases` field in the
    profile's YAML frontmatter. Aliases are used for disambiguation matching.
11. **Provisional profile hygiene.** During each cron run, check for provisional
    profiles older than `max_provisional_age_days` (CONFIG.md, default: 30).
    Flag expired provisionals for operator review.

## Vector DB Protocol [If Enhanced]
- After writing the markdown file, generate an embedding of the file's contents.
- Tag the embedding with `entity_id: [id]`, `type: [type]`, and `status: active`.
- If updating a profile, overwrite the existing embedding matching that `entity_id`.
- If merging profiles, delete the retired profile's embedding and regenerate
  the surviving profile's embedding.

## Profile Template Compliance
You must strictly use the YAML frontmatter and markdown structure defined in
the CRM System Specification (v2.1). Every detail line MUST end with
`[verified: YYYY-MM-DD] [confidence: 0.X]`. Every profile MUST include the
`aliases` field (empty list `[]` if no aliases are known) and the
`## Disambiguation` section.

## File Locking Protocol
- **Single profile update:** Acquire lock on target file → write → release lock.
- **Merge operation:** Acquire lock on both files (alphabetical order by filename
  to prevent deadlocks) → perform merge → release both locks.
- **Lock timeout:** 30 seconds. If lock cannot be acquired, retry up to 3 times
  with exponential backoff, then log an error and skip.
```

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
| Provisional / ambiguous reference *(v2.1)* | 0.30 |

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

### Core Patterns (v2.0)

```regex
/my (name|birthday|job|title|employer|wife|husband|partner|kid|son|daughter|dog|cat|pet) is/i
/i (work|live|moved|started|quit|left|joined) (at|in|to|from)/i
/[A-Z][a-z]+'s birthday/i
/(favorite|favourite|fav) .{1,30} is/i
/my [a-z]+ is named/i
/i (bought|subscribed to) .*/i
```

### Additional Patterns (v2.1)

```regex
/i('m| am) (in|at|visiting|staying in|back in|heading to) [A-Z]/i
/i (started|left|joined|got (hired|fired|laid off)|work(ing)? (at|for)) /i
/i('ve| have) been (into|doing|playing|learning|practicing|watching) /i
/(my|i('m| am)) (health|doctor|diagnosis|surgery|therapy|medication|condition)/i
/i('m| am) (going|traveling|flying|driving|heading) to .{1,40} (next|this|in)/i
/(anniversary|graduation|wedding|retirement|promotion|milestone|born|engaged|divorced)/i
/i (prefer|switched to|don't like|love|hate|stopped using|started using) /i
```

### Pattern Summary (13 Total)

| # | Pattern | Category | Version |
|---|---------|----------|---------|
| 1 | Identity declarations (name, birthday, job, etc.) | Core identity | v2.0 |
| 2 | Life transitions (work, live, moved, etc.) | Life events | v2.0 |
| 3 | Third-party birthdays | Relationships | v2.0 |
| 4 | Favorites/preferences | Preferences | v2.0 |
| 5 | Named relationships ("my X is named") | Relationships | v2.0 |
| 6 | Purchases/subscriptions | Things | v2.0 |
| 7 | Location mentions | Location | v2.1 |
| 8 | Workplace changes | Career | v2.1 |
| 9 | Hobby/interest mentions | Interests | v2.1 |
| 10 | Health updates | Health | v2.1 |
| 11 | Travel plans | Travel | v2.1 |
| 12 | Milestone events | Life events | v2.1 |
| 13 | Preference changes | Preferences | v2.1 |

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

### CONFIG.md Reference

The following parameters are available in the CRM section of `CONFIG.md`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `staleness_threshold_days` | 90 | Days before a fact is considered stale |
| `crm_entity_exclusion_source` | *(none)* | Path to exclusion list (e.g., Breadcrumb recipe.json) |
| `crm_context_budget` | 1000 | Max tokens for CRM profiles in Context Assembler |
| `max_provisional_age_days` | 30 | *(v2.1)* Days before a provisional profile is flagged for review or deletion |

---

## 9. Usage Workflows

### Scenario A: Operator Introduces a New Relationship
1. **Operator:** "Met Dave today. He's a guitarist."
2. **Heartbeat:** Does not hit explicit regex, but is logged to `memory/YYYY-MM-DD.md`.
3. **Cron (06:30):** `crm-miner` reads logs, extracts fact.
4. **Disambiguation:** Searches `Archive/People/` for "Dave" in `name` and `aliases` fields. No match found.
5. **Action:** Creates `Archive/People/dave.md` with relationship: "acquaintance", detail: "guitarist", confidence: 0.6 (inferred/casual), `aliases: []`, `status: canonical`.

### Scenario B: Operator Corrects a Fact
1. **Operator:** "No, Alex Rivera's birthday is March 15th."
2. **Heartbeat:** Hits regex `/birthday is/i`. Flags priority.
3. **Action:** `crm-miner` runs immediately. Acquires lock on `alex-rivera.md`. Moves the old birthday to `## History`. Inserts March 15th with `confidence: 1.0` and today's `last_verified` date. Releases lock.

### Scenario C: Ambiguous Name Reference *(v2.1)*
1. **Operator:** "John told me about this great restaurant."
2. **Cron (06:30):** `crm-miner` reads logs, encounters "John".
3. **Disambiguation Step 1:** Searches for "John" — finds `john-smith.md` (colleague) and `john-doe.md` (neighbor). Both match.
4. **Disambiguation Step 2:** Context clues — "restaurant" doesn't uniquely distinguish. Ambiguous.
5. **Disambiguation Step 3:** Creates `john-provisional.md` with `confidence: 0.3`, `status: provisional`, `provisional_reason: "Ambiguous — could be John Smith (colleague) or John Doe (neighbor). Mentioned a restaurant recommendation."`.
6. **Later:** Operator says "John from work recommended that Italian place." — Context clue "from work" matches `john-smith.md`. CRM Miner updates `john-smith.md` with the restaurant detail. Deletes or merges the provisional profile.

### Scenario D: Profile Merge *(v2.1)*
1. **Operator:** "Oh, that Alex R from Twitter is actually Alex Rivera."
2. **Action:** `crm-miner` identifies `alex-r-twitter.md` and `alex-rivera.md` as the same entity.
3. **Merge Workflow:**
   - `alex-rivera.md` is older/more complete → survives.
   - Acquires locks on both files (alphabetical order).
   - Copies unique details from `alex-r-twitter.md` into `alex-rivera.md`.
   - Adds `Alex R`, `@alexr_twitter` to `alex-rivera.md` aliases.
   - Sets `merged_from: [person_alex-r-twitter]` in disambiguation section.
   - Stubs out `alex-r-twitter.md`: `status: merged`, `merged_into: person_alex-rivera`.
   - Releases both locks.
   - Updates Vector DB (if active).

---

## 10. Build Order

1. **Directories:**
   ```bash
   mkdir -p ~/.openclaw/workspace/Archive/People
   mkdir -p ~/.openclaw/workspace/Archive/Places
   mkdir -p ~/.openclaw/workspace/Archive/Things
   mkdir -p ~/.openclaw/workspace/agents/crm-miner
   ```

2. **Deploy Agent:** Save the SOUL.md from Section 5 into the `crm-miner` directory.

3. **Configuration:** Add the following to your primary `CONFIG.md`:
   ```yaml
   crm_entity_exclusion_source: ~/breadcrumb-trail/recipe.json
   staleness_threshold_days: 90
   max_provisional_age_days: 30
   ```

4. **Heartbeat:** Add all 13 regex patterns from Section 7 into your OpenClaw triage configuration.

5. **Schedule:** Add `30 6 * * * openclaw run crm-miner` to your crontab.

6. **Migrate existing profiles** *(v2.1 upgrade)*: For any profiles created under v2.0, add the following fields:
   - `aliases: []` to YAML frontmatter
   - `## Disambiguation` section with `status: canonical` (for established profiles)
