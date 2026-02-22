# OpenClaw Breadcrumb System — Complete Specification

*Version: 3.0 | Spec ID: `openclaw-breadcrumb-system`*
*Audience: Any OpenClaw operator or agent implementing structured component documentation*
*License: Share freely.*

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Design Principles](#2-design-principles)
3. [Prerequisites](#3-prerequisites)
4. [Core Concepts](#4-core-concepts)
5. [Naming Convention](#5-naming-convention)
6. [Breadcrumb Template](#6-breadcrumb-template)
7. [Type-Specific Extensions](#7-type-specific-extensions)
8. [Where Breadcrumbs Live](#8-where-breadcrumbs-live)
9. [The Bread Recipe (Capabilities Manifest)](#9-the-bread-recipe-capabilities-manifest)
10. [The Breadcrumb Trail (Central Index)](#10-the-breadcrumb-trail-central-index)
11. [Component Hierarchy Rules](#11-component-hierarchy-rules)
12. [The Breadcrumb Baker](#12-the-breadcrumb-baker)
13. [Agent Standing Directive](#13-agent-standing-directive)
14. [Usage Workflows](#14-usage-workflows)
15. [Governance](#15-governance)
16. [Build Order](#16-build-order)

---

## 1. Purpose

### The Problem

Agents build tools, integrations, scripts, and workflows. The next session, they have no memory of what was built, what it does, how to run it, or even that it exists. The operator ends up re-explaining everything, or worse, the agent rebuilds something that already exists.

### The Solution

Every component gets a **breadcrumb** — a structured markdown file that explains exactly what the component is, how it works, and how to use it. All breadcrumbs are cataloged in a **breadcrumb trail** (a central index). A **bread recipe** (capabilities manifest) tells every agent what tools and integrations are available to them before they even start a task. A **Breadcrumb Baker** agent runs periodically to discover undocumented components, validate existing breadcrumbs, and keep the trail current.

### What This Spec Does Not Cover

- Persistent memory, identity evolution, or relationship intelligence (see the OpenClaw Persistent Memory spec if needed)
- Agent deployment infrastructure
- Task orchestration or project management

---

## 2. Design Principles

1. **Breadcrumbs live with their components** (in-situ). The trail is an index, not a copy.
2. **Every agent breadcrumbs what it builds.** The Baker validates and indexes — it does not retroactively document things it didn't build.
3. **The cold start test:** Could an agent with zero prior context use a component with ONLY the breadcrumb and no other information? If not, the breadcrumb is incomplete.
4. **One source of truth.** No duplicate breadcrumbs. The in-situ file is the canonical version.
5. **Capabilities before tasks.** Every agent checks the bread recipe before starting work to know what tools are available.
6. **Token awareness.** Every breadcrumb carries a token estimate so that any token-budgeted system can plan reads before committing context window space.
7. **Extend, don't couple.** This spec works standalone. If a persistent memory system is also active, integration is handled through an external bridge — not through modifications to this spec.

---

## 3. Prerequisites

### Required

| Component | Purpose | Notes |
|-----------|---------|-------|
| OpenClaw instance | Agent runtime | Any supported platform |
| Filesystem access | Breadcrumb storage | Agents need read/write to workspace directories |
| Cron scheduling | Baker automation | OpenClaw native |
| `jq` | Index querying | Standard Linux utility |

### Optional (Recommended)

| Component | Purpose | Impact If Absent |
|-----------|---------|-----------------|
| NFS / shared storage mount | Multi-agent access to breadcrumbs | Single-machine setups work fine without it |
| Multiple agents | Distributed builds | System works with a single agent |

### Infrastructure Notes

- All file paths in this spec are relative to the agent's home directory (`~/` by default).
- The primary breadcrumb trail directory is `~/breadcrumb-trail/`. Operators may symlink this from any workspace path.
- File paths in examples use `~/` notation. Substitute your actual workspace root.

---

## 4. Core Concepts

| Concept | What It Is | File |
|---------|-----------|------|
| **Breadcrumb** | A structured doc for a single component | `{type}.{name}.crumb.md` |
| **Breadcrumb Trail** | Central index of all breadcrumbs | `index.json` + `trail.md` |
| **Bread Recipe** | Capabilities manifest — what can agents do | `recipe.json` |
| **Breadcrumb Baker** | Agent that discovers, validates, indexes | Runs on schedule |
| **Orphan** | A breadcrumb for a component with no home directory | Lives in `orphans/` |

### Component Types

| Type | What It Covers | Examples |
|------|---------------|----------|
| `script` | Shell, Python, Node scripts | Data processors, downloaders, utilities |
| `api` | External API integrations | Gmail API, Telegram auth, Discord gateway |
| `tool` | Standalone applications | Dashboards, web apps, scanners |
| `skill` | OpenClaw skills | Weather skill, email skill |
| `workflow` | Multi-step processes chaining components | Data pipelines, morning briefs |
| `config` | System configurations | Cron jobs, env files, service configs |
| `agent` | Agent definitions and prompts | Any deployed OpenClaw agent |

---

## 5. Naming Convention

### Rules

- **Breadcrumb ID:** `{type}_{component-name}` — underscores between type and name, hyphens within the component name
- **Filename:** `{type}.{component-name}.crumb.md` — dots between segments, `.crumb.md` extension
- **Directory slug:** `{component-name}/` — hyphens only, lowercase

The `.crumb.md` extension is mandatory. It makes breadcrumbs instantly identifiable in any directory listing, `find` command, or grep. It's still valid markdown — any editor renders it fine.

### Examples

| Component | Breadcrumb ID | Filename |
|-----------|--------------|----------|
| Chat Miner script | `script_chat-miner` | `script.chat-miner.crumb.md` |
| Telegram auth API | `api_telegram-auth` | `api.telegram-auth.crumb.md` |
| Dashboard tool | `tool_dashboard` | `tool.dashboard.crumb.md` |
| Data pipeline workflow | `workflow_data-pipeline` | `workflow.data-pipeline.crumb.md` |
| Gmail access integration | `api_gmail-access` | `api.gmail-access.crumb.md` |
| Content rater script | `script_content-rater` | `script.content-rater.crumb.md` |
| Bot agent | `agent_bot-name` | `agent.bot-name.crumb.md` |
| Morning brief workflow | `workflow_morning-brief` | `workflow.morning-brief.crumb.md` |

---

## 6. Breadcrumb Template

Every breadcrumb follows this exact structure. Sections marked REQUIRED must always be present. Sections marked OPTIONAL are included only when relevant.

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
deps: [python3, requests, some-api-token]
---

# Purpose                                    ← REQUIRED
One sentence. What this component does. If you can't say it in one sentence,
you don't understand it yet.

# How It Works                               ← REQUIRED
Numbered steps of the actual flow. Not theory — what literally happens when
this component runs.

1. Step one
2. Step two
3. Step three

# Usage                                      ← REQUIRED
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
| TARGET_ID | 1234567890 | The target identifier |

## Outputs
| Output | Location | Format |
|--------|----------|--------|
| results.json | ~/data/results.json | JSON array |

## Examples                                  ← OPTIONAL
\```bash
# Example with flags
python3 example-tool.py --verbose

# Dry run (no writes)
python3 example-tool.py --dry-run
\```

# Configuration                             ← OPTIONAL (only if config values exist)
| Setting | Value | Notes |
|---------|-------|-------|
| Rate Limit | 10 req/sec | API enforced |
| Run Schedule | Hourly at :30 | Via cron |

# Gotchas                                   ← REQUIRED
Failure modes, rate limits, quirks, things that have broken before.
- ⚠️ Known issue or warning
- 📌 Important constraint or limitation
- 🔧 Maintenance note or workaround

# Related                                   ← OPTIONAL
- [api.some-service.crumb.md](../path/to/api.some-service.crumb.md)

# Changelog                                 ← REQUIRED
Reverse chronological. Every meaningful change gets a line.
- **YYYY-MM-DD**: Description of change
- **YYYY-MM-DD**: Created
```

### YAML Frontmatter Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `id` | Yes | string | Unique breadcrumb ID: `{type}_{component-name}` |
| `name` | Yes | string | Human-readable name |
| `type` | Yes | enum | `script`, `api`, `tool`, `skill`, `workflow`, `config`, `agent` |
| `status` | Yes | enum | `active`, `experimental`, `deprecated`, `broken` |
| `created` | Yes | date | YYYY-MM-DD |
| `updated` | Yes | date | YYYY-MM-DD — update on every change |
| `owner` | Yes | string | Agent name, `shared`, or operator name |
| `source` | Yes | path | Absolute path to the component's directory or primary file |
| `token_estimate` | Yes | integer | Approximate token count of this breadcrumb (Baker calculates automatically) |
| `tags` | No | list | Searchable keywords: `[discord, profiles, cron]` |
| `deps` | No | list | Dependencies: `[python3, telethon, discord-bot-token]` |
| `cron` | No | string | Cron expression if scheduled: `"30 * * * *"` |
| `mount_required` | No | path | If component lives on a network mount: `/mnt/shared-drive` |

---

## 7. Type-Specific Extensions

These sections are added to the standard template when the component type requires them.

### API Breadcrumbs — Additional Sections

```markdown
# Auth
| Field | Value |
|-------|-------|
| Method | OAuth 2.0 / Bearer token / API key |
| Credentials Location | ~/.secrets/service-creds.json |
| Token Refresh | Auto via refresh_token / Manual |
| Scopes | scope.readonly, scope.write |

# Endpoints
| Method | Path | Description |
|--------|------|-------------|
| GET | /resource/list | List resources |
| POST | /resource/create | Create resource |

# Rate Limits
| Limit | Value | Recovery |
|-------|-------|----------|
| Requests/sec | 10 | Auto-retry after 1s |
| Daily quota | 1,000,000 | Resets at midnight |
```

### Script Breadcrumbs — Additional Fields

```yaml
# Add to YAML frontmatter:
language: python3 | bash | node
requires_root: false
cron: "30 * * * *"
```

### Workflow Breadcrumbs — Additional Sections

```markdown
# Steps
| Order | Component | Breadcrumb ID | Input | Output | Depends On |
|-------|-----------|---------------|-------|--------|------------|
| 1 | download.sh | script_download | Source URLs | ~/data/raw/*.mp3 | — |
| 2 | process.py | script_process | *.mp3 files | ~/data/text/*.txt | Step 1 |
| 3 | analyze.py | script_analyze | *.txt files | ~/data/output/*.json | Step 2 |

# Trigger
Cron: `0 6 * * *` (daily at 6am)
Manual: `./run-pipeline.sh`

# End-to-End Runtime
~45 minutes for typical workload

# Failure Handling
- If Step 1 fails: Pipeline aborts. No partial downloads.
- If Step 2 fails: Retry 3x with 30s delay. Skip file on 4th failure.
- If Step 3 fails: Output partial results. Flag in baker.log.
```

### Agent Breadcrumbs — Additional Sections

```markdown
# Agent Configuration
| Field | Value |
|-------|-------|
| Model | model-name-and-version |
| System Prompt | ~/agents/agent-name/SOUL.md |
| MCP Tools | tool-1, tool-2, tool-3 |
| Memory | ~/agents/agent-name/memory/ |
| Host | Platform / VM / Cloud |

# Capabilities
What this agent is authorized to do:
- Read/write files in ~/workspace/
- Execute scripts in ~/scripts/
- Access specific API (read-only)
- NOT authorized to: [explicit denials]

# Communication
- Receives tasks via: [channel]
- Reports results to: [channel]
- Escalates to operator via: [channel]
```

### Tool Breadcrumbs — Additional Sections (for multi-file apps)

```markdown
# Internal Components
| Component | File | Purpose | Own Breadcrumb? |
|-----------|------|---------|----------------|
| fetcher.py | scripts/fetcher.py | Pulls data | Yes |
| parser.py | scripts/parser.py | Structures data | Yes |
| handler.py | routes/handler.py | Route handler | No (internal) |

# Web Interface
| Field | Value |
|-------|-------|
| URL | http://localhost:PORT |
| Port | PORT |
| Framework | Framework name |
| Auth | None (local only) / method |
```

---

## 8. Where Breadcrumbs Live

### Layer 1: In-Situ Breadcrumbs (Source of Truth)

Every component's breadcrumb lives in its own directory, next to the code:

```
~/agents/example-bot/
├── main.py
├── run.sh
├── .env
└── script.example-bot.crumb.md         ← breadcrumb lives here
```

```
~/tools/dashboard/
├── app.py
├── routes/
├── scripts/
├── tool.dashboard.crumb.md             ← parent breadcrumb
└── internals/
    ├── script.fetcher.crumb.md         ← child breadcrumb (independently callable)
    └── script.parser.crumb.md
```

```
~/workflows/data-pipeline/
├── download.sh
├── process.py
├── analyze.py
├── workflow.data-pipeline.crumb.md     ← orchestration breadcrumb
├── script.download.crumb.md            ← step 1
├── script.process.crumb.md             ← step 2
└── script.analyze.crumb.md             ← step 3
```

### Layer 2: Central Hub (Index + Orphans)

The breadcrumb trail directory is the central hub. It contains the index, the recipe, and orphaned breadcrumbs — but NOT copies of in-situ breadcrumbs.

```
~/breadcrumb-trail/
├── index.json                          ← machine-searchable catalog of ALL breadcrumbs
├── recipe.json                         ← capabilities manifest (the Bread Recipe)
├── trail.md                            ← human-readable summary table
├── schema.crumb.md                     ← THIS SPEC (the system eats its own dogfood)
├── baker.log                           ← Breadcrumb Baker run history
├── orphans/                            ← breadcrumbs for components with no home directory
│   ├── api.service-auth.crumb.md       ← external API, no local code dir
│   ├── api.email-access.crumb.md       ← integration config, no standalone dir
│   └── config.cron-jobs.crumb.md       ← system-level config
└── archive/                            ← deprecated breadcrumbs past retention period
```

### What Goes Where

| Scenario | Breadcrumb Location |
|----------|-------------------|
| Script in ~/scripts/ | In-situ: `~/scripts/{name}/script.{name}.crumb.md` |
| Tool in ~/tools/ | In-situ: `~/tools/{name}/tool.{name}.crumb.md` |
| Agent in ~/agents/ | In-situ: `~/agents/{name}/agent.{name}.crumb.md` |
| External API (no local code) | Orphan: `~/breadcrumb-trail/orphans/api.{name}.crumb.md` |
| System config (cron, env) | Orphan: `~/breadcrumb-trail/orphans/config.{name}.crumb.md` |
| Network-mounted project | In-situ at mount + fallback orphan copy |
| OpenClaw skill | In-situ: `~/.openclaw/skills/{name}/skill.{name}.crumb.md` |

### Network-Mounted Resources

When a component lives on a network share (e.g., `/mnt/shared/projects/example/`):

1. The in-situ breadcrumb lives at the mount: `/mnt/shared/projects/example/workflow.example.crumb.md`
2. A fallback copy lives in orphans: `~/breadcrumb-trail/orphans/workflow.example.crumb.md`
3. The index entry includes `mount_required` and `fallback` fields
4. The Baker checks mount availability during validation and flags when mounts are offline

---

## 9. The Bread Recipe (Capabilities Manifest)

### What It Is

The Bread Recipe is a JSON manifest that answers one question: **"What can I do?"**

Every agent's system prompt should include the instruction to check the recipe before starting any task. The recipe is NOT the same as the index — the index tells you where things are; the recipe tells you what capabilities are available and what they're for.

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
      "summary": "Read/write calendar events. Can create and modify, cannot delete.",
      "permissions": ["read", "create", "modify"],
      "restrictions": ["no-delete"]
    },
    "data_pipeline": {
      "status": "active",
      "breadcrumb_id": "workflow_data-pipeline",
      "breadcrumb_path": "~/workflows/data-pipeline/workflow.data-pipeline.crumb.md",
      "summary": "Processes data end-to-end. Runs daily. Manual trigger: ./run-pipeline.sh",
      "permissions": ["execute"],
      "restrictions": ["resource-intensive"]
    }
  },
  "mounts": {
    "/mnt/shared-drive": {
      "status": "online",
      "last_checked": "YYYY-MM-DDTHH:MM:SSZ",
      "contains": ["workflow_shared-project"]
    }
  }
}
```

### Recipe vs. Index

| Aspect | Recipe (`recipe.json`) | Index (`index.json`) |
|--------|----------------------|---------------------|
| Question answered | "What can I do?" | "Where is everything?" |
| Audience | Agents starting a task | Agents/humans searching for a component |
| Content | Capabilities, permissions, restrictions | IDs, paths, metadata |
| Updated by | Baker + manual when new integrations added | Baker on every run |
| Checked | Before every task | When searching for a specific component |

---

## 10. The Breadcrumb Trail (Central Index)

### index.json

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
    },
    {
      "id": "api_email-access",
      "name": "Email Access",
      "type": "api",
      "status": "active",
      "path": "~/breadcrumb-trail/orphans/api.email-access.crumb.md",
      "source": null,
      "token_estimate": 280,
      "tags": ["email", "oauth"],
      "owner": "shared",
      "created": "YYYY-MM-DD",
      "updated": "YYYY-MM-DD",
      "mount_required": null,
      "fallback": null
    },
    {
      "id": "workflow_shared-project",
      "name": "Shared Project",
      "type": "workflow",
      "status": "active",
      "path": "/mnt/shared-drive/projects/example/workflow.shared-project.crumb.md",
      "source": "/mnt/shared-drive/projects/example/",
      "token_estimate": 420,
      "tags": ["collaboration"],
      "owner": "shared",
      "created": "YYYY-MM-DD",
      "updated": "YYYY-MM-DD",
      "mount_required": "/mnt/shared-drive",
      "fallback": "~/breadcrumb-trail/orphans/workflow.shared-project.crumb.md"
    }
  ],
  "tags_index": {
    "data": ["script_example"],
    "email": ["api_email-access"],
    "cron": ["script_example"],
    "collaboration": ["workflow_shared-project"]
  }
}
```

### trail.md

Human-readable summary for quick scanning:

```markdown
# Breadcrumb Trail
Last updated: YYYY-MM-DD HH:MM UTC | Total: N breadcrumbs

## Active Components

| ID | Name | Type | Owner | Tokens | Path |
|----|------|------|-------|--------|------|
| script_example | Example Script | script | agent-name | 350 | ~/scripts/example/ |
| api_email-access | Email Access | api | shared | 280 | orphans/ |

## Experimental

| ID | Name | Type | Owner | Tokens | Path |
|----|------|------|-------|--------|------|
| (none) | | | | | |

## Deprecated

| ID | Name | Deprecated On | Reason |
|----|------|--------------|--------|
| (none) | | | |
```

---

## 11. Component Hierarchy Rules

### The Independently Callable Rule

> **If a component can be invoked on its own (by a user, cron job, another script, or another agent), it gets its own breadcrumb. If it's only called internally by a parent component, it gets documented in the parent's breadcrumb.**

### Examples

**Multi-file web app:**

```
~/tools/dashboard/
├── app.py                                 ← internal, no own crumb
├── routes/
│   ├── main.py                            ← internal, no own crumb
│   └── settings.py                        ← internal, no own crumb
├── scripts/
│   ├── fetch-data.py                      ← independently callable → OWN CRUMB
│   └── generate-report.py                 ← independently callable → OWN CRUMB
├── tool.dashboard.crumb.md                ← PARENT crumb (documents the whole app + internal modules)
└── internals/
    ├── script.fetch-data.crumb.md
    └── script.generate-report.crumb.md
```

The parent crumb's "Internal Components" section lists ALL modules — both those with their own crumbs and those documented only in the parent.

**Workflow with shared steps:**

Each step script gets its own crumb because it can be run independently. The workflow crumb documents the chain — the orchestration layer.

**Single-file script:**

One script, one crumb, same directory. Simple.

```
~/scripts/cleanup-logs/
├── cleanup-logs.sh
└── script.cleanup-logs.crumb.md
```

---

## 12. The Breadcrumb Baker

### Overview

The Baker is a dedicated OpenClaw agent that maintains the Breadcrumb System. It discovers undocumented components, validates existing breadcrumbs, and rebuilds the central index.

**Name:** `breadcrumb-baker`
**Location:** `agents/breadcrumb-baker/`
**Schedule:** See Baker Schedule table below

### SOUL.md

```markdown
# Breadcrumb Baker

## Role
Documentation Infrastructure Maintenance Agent

## Mission
Maintain the Breadcrumb System — a structured documentation infrastructure that
ensures every component in this environment is documented, validated, and
discoverable.

Your canonical reference is ~/breadcrumb-trail/schema.crumb.md — always follow
that spec for breadcrumb format, naming, and placement.

## Operating Rhythm

You run on a schedule (see below). Each run has three phases executed in order.

---

### PHASE 1: DISCOVER (Find undocumented components)

Scan these directories for files that do NOT have a corresponding .crumb.md file
in the same directory:

  ~/agents/       → look for *.py, *.sh, *.js, package.json, SOUL.md
  ~/scripts/      → look for *.py, *.sh, *.js
  ~/tools/        → look for anything executable, app.py, server.js, main.*
  ~/skills/       → look for skill manifests, SKILL.md
  ~/.openclaw/    → look for config files, workspace items, integrations
  ~/workflows/    → look for multi-file directories with pipeline scripts

For each discovery:
- Log it in ~/breadcrumb-trail/baker.log
- DO NOT auto-generate breadcrumbs for components you did not build in this session
- Flag them as "NEEDS BREADCRUMB" in the log

Auto-draft exception: You may create a DRAFT breadcrumb ONLY if the discovered
component has a README.md, SKILL.md, or responds to `--help`. Auto-drafted
breadcrumbs always get:
- status: experimental
- A Gotcha entry: "⚠️ Auto-generated by Baker — not validated by builder"
- Placement in ~/breadcrumb-trail/orphans/ (not in-situ)

If none of these signals exist, flag-only. Do not guess.

### PHASE 2: VALIDATE (Check existing breadcrumbs)

For every .crumb.md file found in the filesystem:

1. Parse the YAML frontmatter
2. Check if the `source` path still exists
3. Check if files in the source directory have been modified since the breadcrumb's
   `updated` date
4. Check if any mount_required paths are accessible
5. Calculate `token_estimate` (approximate token count of the full breadcrumb file)
   and update the YAML frontmatter if the estimate has changed by more than 10%

Actions:
- Source path GONE → move breadcrumb to orphans/, set status: deprecated,
  add deprecation note to changelog
- Source MODIFIED since last update → flag as STALE in baker.log
  (do NOT auto-modify breadcrumb content — flag for human/agent review)
- Mount OFFLINE → update recipe.json mount status, log warning
- Breadcrumb has status: broken → check if source has been fixed, flag if so

### PHASE 3: INDEX (Rebuild the trail)

1. Walk ALL .crumb.md files across the entire filesystem (~/agents, ~/scripts,
   ~/tools, ~/workflows, ~/skills, ~/.openclaw, ~/breadcrumb-trail/orphans/)
2. Parse YAML frontmatter from each
3. Rebuild ~/breadcrumb-trail/index.json:
   - All breadcrumb entries with id, name, type, status, path, source, tags,
     owner, token_estimate, created, updated
   - Rebuild the tags_index mapping
4. Rebuild ~/breadcrumb-trail/trail.md:
   - Human-readable summary table grouped by status (active, experimental,
     deprecated, broken)
   - Include token_estimate column
5. Validate ~/breadcrumb-trail/recipe.json:
   - Check that every capability still has a valid breadcrumb
   - Flag capabilities whose breadcrumbs are deprecated or broken
6. Log stats to baker.log

---

### What You CAN Auto-Create

- Breadcrumbs for components YOU just built in the current session
- Breadcrumbs when explicitly asked by the operator or another authorized user
- Draft breadcrumbs (status: experimental) when the component has a README.md,
  SKILL.md, or --help output
- Index and trail updates on every run
- Recipe updates when capabilities change

### What You CANNOT Auto-Create

- Breadcrumbs for pre-existing components that lack README/SKILL/--help
  (you lack the context to document them accurately — flag instead)
- Breadcrumbs from chat history or memory alone
  (too much risk of hallucination — flag for review)

### Quality Standards

Every breadcrumb you create must pass the COLD START TEST:

  "Could an agent with zero prior context successfully use this component
   with ONLY the breadcrumb and no other information?"

If the answer is no, the breadcrumb is incomplete. Specifically:
- Commands must be copy-paste ready with real paths and real values
- Credentials: document WHERE they are stored, NEVER the actual values
- Document failure modes and known workarounds in Gotchas
- If you don't know something, write "UNKNOWN — needs investigation" —
  never guess

### Log Format

Append to ~/breadcrumb-trail/baker.log on every run:

  === BAKER RUN: {timestamp} | Type: {scheduled|manual|deep} ===
  PHASE 1 — DISCOVER
    Found {N} undocumented components:
    - ~/scripts/new-tool.py (NEEDS BREADCRUMB)
    - ~/agents/helper-bot/ (NEEDS BREADCRUMB)
    Created {N} draft breadcrumbs:
    - ~/breadcrumb-trail/orphans/script.new-tool.crumb.md (from README.md)
  PHASE 2 — VALIDATE
    Checked {N} breadcrumbs:
    - {N} OK
    - {N} STALE (source modified since last update)
    - {N} DEPRECATED (source removed)
    - {N} MOUNT OFFLINE
    - {N} token_estimate updated
  PHASE 3 — INDEX
    Total breadcrumbs: {N}
    Active: {N} | Experimental: {N} | Deprecated: {N} | Broken: {N}
    Recipe capabilities: {N} active, {N} flagged
  === END BAKER RUN ===
```

### Baker Schedule

| Run Type | Frequency | What It Does |
|----------|-----------|-------------|
| **Standard scan** | Every 6 hours | Phase 1 (Discover) + Phase 2 (Validate) + Phase 3 (Index) |
| **Quick reindex** | On demand / after any build | Phase 3 only (Index rebuild) |
| **Post-build** | Triggered after any agent builds something | Validate new crumb + reindex |

Default standard scan times: 05:00, 11:00, 17:00, 23:00. Adjust as needed. When running alongside other cron-scheduled systems, ensure Baker does not overlap with their write windows.

### Baker Inputs / Outputs

| Direction | Source/Destination | Type | Notes |
|-----------|--------------------|------|-------|
| **Reads** | `~/agents/`, `~/scripts/`, `~/tools/`, `~/workflows/`, `~/skills/`, `~/.openclaw/` | Filesystem | Component discovery |
| **Reads** | All `*.crumb.md` files | Files | Validation and indexing |
| **Reads** | `~/breadcrumb-trail/recipe.json` | JSON | Capability validation |
| **Writes** | `~/breadcrumb-trail/index.json` | JSON | Rebuilt every run |
| **Writes** | `~/breadcrumb-trail/trail.md` | Markdown | Rebuilt every run |
| **Writes** | `~/breadcrumb-trail/baker.log` | Log | Appended every run |
| **Writes** | `~/breadcrumb-trail/recipe.json` | JSON | Mount status, capability flags |
| **Writes** | `*.crumb.md` files | Files | token_estimate updates |
| **Writes** | `~/breadcrumb-trail/orphans/` | Files | Deprecated crumbs, auto-drafts |

---

## 13. Agent Standing Directive

Every agent in the system must include this directive in their system prompt or operating instructions. This is not a suggestion — it is a hard behavioral gate. Agents that skip these steps will rebuild things that exist, break things that are documented, and waste operator time.

```
### BREADCRUMB DIRECTIVE

You are part of the Breadcrumb System. These rules are mandatory, not advisory.

═══════════════════════════════════════════════════════════════════
RULE 1: PRE-FLIGHT CHECK (MANDATORY — NEVER SKIP)
═══════════════════════════════════════════════════════════════════

BEFORE executing any task that involves external services, APIs, tools,
or any component you did not build in THIS session:

1. Read ~/breadcrumb-trail/recipe.json
2. Identify which capabilities are needed for the task
3. For each needed capability, read the .crumb.md file at the path
   listed in recipe.json
4. THEN proceed with the task

Do not skip this. Do not rely on memory. Do not assume you know how
something works. The breadcrumb is always more accurate than your recall.

If you are operating within a token-budgeted context system, check the
breadcrumb's token_estimate before loading it. Prioritize recipe.json
(always load), then load individual crumbs only when the task requires
a specific component and the budget permits.

═══════════════════════════════════════════════════════════════════
RULE 2: SEARCH BEFORE YOU ASK
═══════════════════════════════════════════════════════════════════

When you are unsure how something works or where something is:

1. FIRST: Search index.json for the component:
   jq '.breadcrumbs[] | select(.name | test("keyword"; "i"))' ~/breadcrumb-trail/index.json
2. Read the matching .crumb.md file
3. ONLY IF the breadcrumb does not exist or says "UNKNOWN": ask the operator
4. If you had to ask the operator, CREATE or UPDATE the breadcrumb with
   the answer so you never have to ask again

This turns every question into a breadcrumb. The system gets smarter
every time you hit a gap.

═══════════════════════════════════════════════════════════════════
RULE 3: HOW TO QUERY THE SYSTEM
═══════════════════════════════════════════════════════════════════

To find a capability by keyword:
  jq '.capabilities | to_entries[] | select(.value.summary | test("keyword"; "i"))' ~/breadcrumb-trail/recipe.json

To find a component by name:
  jq '.breadcrumbs[] | select(.name | test("keyword"; "i"))' ~/breadcrumb-trail/index.json

To find components by tag:
  jq '.tags_index.tagname' ~/breadcrumb-trail/index.json

To find components by type:
  jq '.breadcrumbs[] | select(.type == "workflow")' ~/breadcrumb-trail/index.json

To find breadcrumbs under a token budget:
  jq '.breadcrumbs[] | select(.token_estimate < 500)' ~/breadcrumb-trail/index.json

To find all breadcrumbs on disk:
  find ~ -name "*.crumb.md" -not -path "*/node_modules/*"

The recipe IS your memory of what you can do.
The index IS your memory of where things are.
These queries ARE your conversational lookup. Use them.

═══════════════════════════════════════════════════════════════════
RULE 4: WHEN YOU BUILD SOMETHING NEW
═══════════════════════════════════════════════════════════════════

- Create a .crumb.md file in the component's directory immediately
- Follow the schema at ~/breadcrumb-trail/schema.crumb.md
- Use the naming convention: {type}.{component-name}.crumb.md
- Set token_estimate to 0 (the Baker will calculate it on the next run)
- Update ~/breadcrumb-trail/recipe.json if this is a new capability
- Include: status, summary, permissions, restrictions
- The Baker will index it on the next run

═══════════════════════════════════════════════════════════════════
RULE 5: WHEN YOU MODIFY SOMETHING EXISTING
═══════════════════════════════════════════════════════════════════

- Update the component's .crumb.md file
- Update the `updated` date in YAML frontmatter
- Add a changelog entry

═══════════════════════════════════════════════════════════════════
RULE 6: WHEN SOMETHING BREAKS
═══════════════════════════════════════════════════════════════════

- Add the failure to the breadcrumb's Gotchas section
- If you fix it, document the fix
- If you can't fix it, set status to `broken` in the YAML frontmatter

═══════════════════════════════════════════════════════════════════
RULE 7: COLD START TEST
═══════════════════════════════════════════════════════════════════

Before finalizing any breadcrumb, ask: "Could an agent with zero context
use this with ONLY the breadcrumb?" If no, add what's missing.
```

---

## 14. Usage Workflows

### Workflow A: Agent Needs to Use a Forgotten Tool

```
1. Agent receives task: "Send a summary email of today's activity"
2. Agent reads recipe.json → sees "email" capability is active
3. Agent reads api.email-access.crumb.md → learns auth method, endpoints, restrictions
4. Agent checks token_estimate before loading additional crumbs if token-budgeted
5. Agent executes the task using documented capabilities
6. Agent follows restriction: "no-auto-send" → presents draft for approval
```

### Workflow B: Agent Builds Something New

```
1. Agent builds a new tool at ~/tools/status-board/
2. Agent creates tool.status-board.crumb.md in that directory (token_estimate: 0)
3. Agent updates recipe.json with new "status_board" capability
4. On next Baker run: Baker discovers the crumb, validates it, calculates
   token_estimate, adds to index
5. Next session: any agent can find and use status-board via recipe or index
```

### Workflow C: Something Breaks

```
1. Agent tries to run a pipeline → a step fails
2. Agent checks the workflow crumb → Gotchas section shows known issues
3. Issue is new → Agent adds it to Gotchas with a timestamp
4. Agent fixes the issue → adds the fix to Gotchas and a changelog entry
5. Agent updates `updated` date in frontmatter
```

### Workflow D: Operator Asks "What Can You Do?"

```
1. Agent reads recipe.json
2. Lists all active capabilities with summaries
3. For each, can drill into the breadcrumb for full details
4. No guessing, no hallucination — only documented capabilities
```

### Workflow E: Searching for a Component

```bash
# By name (in index)
cat ~/breadcrumb-trail/index.json | jq '.breadcrumbs[] | select(.name | test("example"; "i"))'

# By tag
cat ~/breadcrumb-trail/index.json | jq '.tags_index.email'

# By type
cat ~/breadcrumb-trail/index.json | jq '.breadcrumbs[] | select(.type == "workflow")'

# By token budget (find crumbs under 500 tokens)
cat ~/breadcrumb-trail/index.json | jq '.breadcrumbs[] | select(.token_estimate < 500)'

# Filesystem search (finds all crumbs)
find ~ -name "*.crumb.md" -not -path "*/node_modules/*"

# Quick grep across all crumbs
grep -rl "keyword" ~/breadcrumb-trail/ ~/agents/ ~/scripts/ ~/tools/ --include="*.crumb.md"
```

---

## 15. Governance

### Ownership

- **System Owner:** Operator — can request changes to any breadcrumb, approve major components
- **Maintainers:** All deployed agents — create and update breadcrumbs for what they build
- **Baker:** Validates and indexes — does not own content

### Status Lifecycle

```
experimental → active → deprecated → archived
                 ↓
              broken → active (when fixed)
                 ↓
              deprecated (when abandoned)
```

| Status | Meaning | Action |
|--------|---------|--------|
| `experimental` | New, untested, may change | Use with caution |
| `active` | Stable, tested, documented | Use freely |
| `broken` | Known failure, not currently working | Do not use — check Gotchas for details |
| `deprecated` | Replaced or removed, still documented | Do not use — check for replacement |

### Deprecation Rules

- When a component is deleted → Baker sets breadcrumb status to `deprecated`, moves to orphans if source is gone
- Deprecated breadcrumbs are kept for 90 days
- After 90 days → moved to `~/breadcrumb-trail/archive/` (preserved but excluded from index)
- Archived breadcrumbs can be restored if the component returns

### Quality Reviews

- Baker flags stale breadcrumbs every run
- Operator reviews flagged items periodically (recommended: weekly)
- Any agent can update a breadcrumb it uses if it finds inaccuracies

---

## 16. Build Order

When building the Breadcrumb System from scratch, follow this order:

### Step 1: Create Directory Structure

```bash
mkdir -p ~/breadcrumb-trail/orphans
mkdir -p ~/breadcrumb-trail/archive
```

### Step 2: Install This Spec

Save this document as:
```
~/breadcrumb-trail/schema.crumb.md
```

This spec is itself a breadcrumb — the system documents itself.

### Step 3: Create Initial recipe.json

Start with known capabilities. Use the template from Section 9. Even if empty:

```json
{
  "version": "1.0",
  "last_updated": "YYYY-MM-DDTHH:MM:SSZ",
  "capabilities": {},
  "mounts": {}
}
```

### Step 4: Create Initial index.json

```json
{
  "version": "1.0",
  "last_updated": "YYYY-MM-DDTHH:MM:SSZ",
  "total_breadcrumbs": 0,
  "breadcrumbs": [],
  "tags_index": {}
}
```

### Step 5: Create Breadcrumbs for Existing Components

Walk through every known component and create its `.crumb.md` file in-situ. Priority order:

1. APIs and integrations (most likely to be forgotten)
2. Agents (need to know their own capabilities)
3. Workflows (multi-step processes with dependencies)
4. Scripts and tools (individual components)
5. Configurations (system-level settings)

### Step 6: Populate recipe.json

For every API, integration, and capability that agents can use, add an entry to recipe.json.

### Step 7: Deploy the Baker Agent

Deploy the Breadcrumb Baker with its SOUL.md from Section 12. Execute manually to:
- Discover any components missed in Step 5
- Validate all new breadcrumbs
- Calculate token_estimate for all breadcrumbs
- Build the full index.json and trail.md

### Step 8: Add Standing Directive to All Agents

Update every agent's system prompt with the Breadcrumb Directive from Section 13.

### Step 9: Schedule the Baker

Set up cron jobs per the schedule in Section 12:
- Standard scan: every 6 hours (default: 05:00, 11:00, 17:00, 23:00)

### Step 10: Verify Cold Start

Test: Spin up a fresh agent session with no prior context. Give it a task that requires using an existing component. The agent should be able to find the component via recipe.json → read its breadcrumb → execute successfully.

If it can't, the breadcrumb is incomplete. Fix it and re-test.

---

## Summary

| Aspect | Value |
|--------|-------|
| **Breadcrumb format** | YAML frontmatter + Markdown body, `.crumb.md` extension |
| **Source of truth** | In-situ (lives with the component) |
| **Central hub** | `~/breadcrumb-trail/` (index + recipe + orphans) |
| **Capabilities manifest** | `recipe.json` — checked before every task |
| **Searchable index** | `index.json` — rebuilt by Baker every run |
| **Token awareness** | `token_estimate` in every breadcrumb and index entry |
| **Baker schedule** | Every 6 hours (standard), on-demand (post-build) |
| **Hierarchy rule** | Independently callable → own crumb; internal → documented in parent |
| **Cold start test** | Can a zero-context agent use this with only the breadcrumb? |
| **Agent directive** | All agents breadcrumb what they build, check recipe before tasks |

---

## Integration Note

This spec is designed to work standalone. If you are also running a persistent memory system (e.g., the OpenClaw Persistent Memory spec), a separate Integration Bridge spec provides the configuration that connects the two systems without modifying either. The Bridge handles unified cron scheduling, token budget allocation, heartbeat pattern registration, CRM exclusion rules, and recall failure routing.

---

*This specification is itself a breadcrumb. It lives at `~/breadcrumb-trail/schema.crumb.md` and is the canonical reference for the entire system.*

---

## Gap Feedback Loop (Optional Enhancement)

The Breadcrumb System captures what's documented, but how do you know what's *missing*? The Feedback Loop detects Gap capability gaps from conversation and ensures they're captured.

### What It Detects

When an operator mentions something the agent doesn't have capability for:
- "can you send an email?"
- "post this to Twitter"
- "why don't you have X feature?"

### Components

| Component | Purpose | Location |
|----------|---------|----------|
| `gaps.md` | Log of detected gaps | `~/.openclaw/gaps/gaps.md` |
| `detect-gap.sh` | Script to check for gaps | `~/.openclaw/gaps/detect-gap.sh` |
| CONFIG.md patterns | Heartbeat triggers | In your CONFIG.md |

### How It Works

1. **Detect** — Heartbeat runs `detect-gap.sh` on each poll
2. **Log** — If gap found, append to `gaps.md` as OPEN
3. **Resolve** — When capability is added, mark gap as RESOLVED
4. **Verify** — On next run, Baker confirms gap is addressed

### Setup

```bash
# Create gaps directory
mkdir -p ~/.openclaw/gaps

# Create gaps log
cat > ~/.openclaw/gaps/gaps.md << 'MD'
# Gap Feedback Log

| ID | Capability | Status | Detected | Resolved |
|----|------------|--------|----------|----------|
MD

# Create detection script
cat > ~/.openclaw/gaps/detect-gap.sh << 'SCRIPT'
#!/bin/bash
GAP_LOG="$HOME/.openclaw/gaps/gaps.md"
INDEX="$HOME/.openclaw/breadcrumb-trail/index.json"

# Check for new gaps (customize patterns as needed)
PATTERNS=(
  "can you send"
  "can you post"
  "can you make"
  "i need to"
)

# Log new gaps to gaps.md
echo "Checking for gaps..."
SCRIPT
chmod +x ~/.openclaw/gaps/detect-gap.sh
```

### Integration with Heartbeat

Add to your HEARTBEAT.md:
```markdown
## Gap Detection

bash ~/.openclaw/gaps/detect-gap.sh
```

### Gap States

| State | Meaning |
|-------|---------|
| OPEN | Capability mentioned but not documented |
| IN_PROGRESS | Being addressed |
| RESOLVED | Capability documented via breadcrumb |

### Example

| ID | Capability | Status | Detected | Resolved |
|----|------------|--------|----------|----------|
| GAP-001 | Slack integration | RESOLVED | 2026-02-22 | 2026-02-22 |
| GAP-002 | Send SMS | OPEN | 2026-02-22 | — |

---

