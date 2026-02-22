# OpenClaw Agent Creation & Registration Spec

*Version: 3.0 | Spec ID: `openclaw-agent-creation`*
*Audience: Any OpenClaw operator creating new specialist agents*
*License: Share freely. Customize per agent.*
*Changelog: [See Section 21](#21-changelog)*

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Design Principles](#2-design-principles)
3. [Agent Taxonomy & Naming](#3-agent-taxonomy--naming)
4. [Prerequisites](#4-prerequisites)
5. [System Architecture](#5-system-architecture)
6. [The Two Paths](#6-the-two-paths)
7. [Proper Agent Creation — Path A](#7-proper-agent-creation--path-a)
8. [Bulk Sync — Path B](#8-bulk-sync--path-b)
9. [Action Tracking (Async Telemetry)](#9-action-tracking-async-telemetry)
10. [Cron Setup](#10-cron-setup)
11. [Agent Folder Structure](#11-agent-folder-structure)
12. [File Manifest](#12-file-manifest)
13. [AGENT_CONFIG.md Template](#13-agent_configmd-template)
14. [Agent Lifecycle Management](#14-agent-lifecycle-management)
15. [Health Checks & Monitoring](#15-health-checks--monitoring)
16. [Security Considerations](#16-security-considerations)
17. [Multi-Agent Coordination](#17-multi-agent-coordination)
18. [Troubleshooting](#18-troubleshooting)
19. [Build Order](#19-build-order)
20. [Quick Reference](#20-quick-reference)
21. [Changelog](#21-changelog)

---

## 1. Purpose

This spec defines how to create OpenClaw agents that are fully functional, trackable, and maintainable. It solves the **"zombie agent" problem** — agents that exist in folders but never run, show 0 actions, and have no observability.

### What This Spec Delivers

- Agents that appear in the dashboard with accurate action counts
- Cron jobs that trigger on schedule without colliding with other system crons
- Proper observability (status, last run, action history)
- Asynchronous action tracking that records work without blocking the agent
- Seamless integration with the Breadcrumb and Persistent Memory systems
- Lifecycle management from creation through decommissioning
- Health check patterns for detecting silent failures

### What This Spec Does Not Cover

- **Agent behavior** — Each agent's purpose is defined in its own `SOUL.md`
- **Skill development** — Handled by individual agent specs
- **Deployment to production** — Agent-specific
- **LLM selection criteria** — Model choice is outside this spec's scope
- **Breadcrumb Baker internals** — See the Breadcrumb spec for indexing logic

---

## 2. Design Principles

| # | Principle | Rationale |
|---|-----------|-----------|
| 1 | **Registered before scheduled** | An agent must exist in the database before cron can trigger it |
| 2 | **Agnostic observability** | Agents record actions but don't care where they are stored. Telemetry endpoints belong in `.env`, not hardcoded in agent files |
| 3 | **Isolation by default** | Each agent gets its own workspace, session, and state |
| 4 | **Non-blocking telemetry** | Telemetry logs async. A dropped database connection must never crash an agent's main task |
| 5 | **Fail fast, fail loud** | Broken agents should alert, not silently fail |
| 6 | **One way to run** | Cron triggers agent → agent reads Breadcrumbs → agent does work → agent records action |
| 7 | **Idempotent by design** | Running an agent twice should not corrupt state or duplicate work |
| 8 | **Ecosystem alignment** | Agent configs and schedules must not conflict with master system parameters (Integration Bridge, Lab Manager) |
| 9 | **Convention over configuration** | Standard naming, paths, and structures reduce operator error |

---

## 3. Agent Taxonomy & Naming

### Naming Convention

```text
<domain>-<role>
```

- **Lowercase, hyphen-delimited** — no underscores, no camelCase
- **Domain first** — the system, resource, or area the agent owns
- **Role second** — the action archetype that defines how it operates

**Examples:**

| Name | Domain | Role |
|------|--------|------|
| `memory-miner` | memory | miner |
| `breadcrumb-baker` | breadcrumb | baker |
| `crm-miner` | crm | miner |
| `lab-manager` | lab | manager |
| `deployment-auditor` | deployment | auditor |
| `soul-keeper` | soul | keeper |
| `feed-scanner-reddit` | feed-scanner | reddit |

### Agent Roles (Archetypes)

| Role | Description | Typical Schedule |
|------|-------------|------------------|
| **Scanner** | Reads/monitors an external data source | Every 1–6h |
| **Miner** | Extracts and stores structured data from logs | Every 6–24h |
| **Baker** | Discovers, validates, and indexes components | Every 6h |
| **Builder** | Produces artifacts (reports, summaries) | Daily/weekly |
| **Compactor** | Reduces, cleans, or archives data | Weekly |
| **Auditor / Checker** | Validates health, state, or implementation | On-demand / Daily |
| **Keeper / Manager** | Evolves identity, oversees system operations | Daily / Weekly |
| **Dispatcher** | Routes tasks to other specialist agents | Event-driven |

---

## 4. Prerequisites

### Core (Required)

| Component | Purpose | Verification |
|-----------|---------|--------------|
| OpenClaw instance | Agent runtime | `openclaw --version` |
| Workspace directory | Agent files | `ls ~/.openclaw/workspace/agents/` |
| `.env` file | Environment secrets | `cat ~/.openclaw/.env \| grep OPENCLAW_TELEMETRY_API` |
| `openclaw agents add` | Proper agent creation | `openclaw agents add --help` |
| Cron scheduling | Timed execution | `openclaw cron list` |

### Enhanced (Optional — Recommended)

| Component | Purpose | Impact If Absent |
|-----------|---------|------------------|
| `record-action.sh` | Centralized async action logging | Agents must curl manually |
| Monitoring dashboard | Real-time status | Must check cron runs manually |
| `health-checker` agent | Verify agents running | Silent failures go unnoticed |
| Log rotation | Prevent disk exhaustion | Logs grow unbounded |

### Environment Variables

Set these once in `~/.openclaw/.env`. Every command and script in this spec references them — **nothing is hardcoded**.

```bash
# ~/.openclaw/.env — source this in your shell profile
export OPENCLAW_TELEMETRY_API="http://127.0.0.1:3000"   # Telemetry API base URL
export DEFAULT_MODEL="MiniMax-M2.5"            # Default LLM for agent creation
# Add external API keys below as needed
```

All scripts in this spec use `${OPENCLAW_TELEMETRY_API:?}` — they will refuse to run if the variable is unset.

### Pre-Flight Checklist

```bash
# Run before creating any agent
echo "=== OpenClaw Pre-Flight ==="
source ~/.openclaw/.env 2>/dev/null
openclaw --version                          # Runtime exists
curl -sf "${OPENCLAW_TELEMETRY_API:?}/api/agents" > /dev/null && echo "Telemetry API: OK" || echo "Telemetry API: FAIL"
ls ~/.openclaw/workspace/agents/ 2>/dev/null && echo "Workspace: OK" || echo "Workspace: MISSING"
openclaw cron list > /dev/null 2>&1 && echo "Cron system: OK" || echo "Cron system: FAIL"
[ -n "${DEFAULT_MODEL:-}" ] && echo "Default model: ${DEFAULT_MODEL}" || echo "DEFAULT_MODEL: NOT SET"
```

---

## 5. System Architecture

### Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                     AGENT CREATION                            │
│  openclaw agents add <name> --workspace <dir>                │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│         NATIVE REGISTRATION & SCHEDULING                      │
│  • OpenClaw agent config (.json) updated                     │
│  • openclaw cron add --agent <name>                          │
└──────────────────────────┬───────────────────────────────────┘
                           │
═══════════════════════════▼════════════════════════════════════
            EXECUTION LAYER (Isolated Session)
────────────────────────────────────────────────────────────────
│ 1. Agent reads SOUL.md & USER.md
│ 2. Agent reads recipe.json (Breadcrumb Directive)
│ 3. Executes task → Generates output
═══════════════════════════╤════════════════════════════════════
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│         ASYNCHRONOUS TELEMETRY (record-action.sh)             │
│  Agent fires-and-forgets in background:                      │
│  $ ~/.openclaw/scripts/record-action.sh "<agent>" "<type>"   │
│                                                               │
│  • Wrapper sources ~/.openclaw/.env                          │
│  • POSTs to $OPENCLAW_TELEMETRY_API with retry + timeout      │
│  • Falls back to local log if API is unreachable             │
└──────────────────────────────────────────────────────────────┘
```

### Data Flow

```
Cron Timer
  │
  ├─▶ openclaw cron run
  │     │
  │     ├─▶ Spawns isolated session
  │     │     │
  │     │     ├─▶ Loads SOUL.md + USER.md
  │     │     ├─▶ Reads recipe.json (Breadcrumb Directive)
  │     │     ├─▶ Executes task from --message
  │     │     ├─▶ Fires record-action.sh in background (async)
  │     │     │         ├─▶ POST /api/actions (success)
  │     │     │         └─▶ Fallback to local log (if API down)
  │     │     │
  │     │     └─▶ Session terminates immediately (does not wait for telemetry)
  │     │
  │     └─▶ Session cleanup
  │
  └─▶ Next scheduled run
```

---

## 6. The Two Paths

| Criteria | Path A: Proper Creation | Path B: Bulk Sync |
|----------|------------------------|--------------------|
| **Use when** | Production agents needing full lifecycle | Quick prototyping or dashboard visibility |
| **Command** | `openclaw agents add` | `POST /api/agents?action=sync` |
| **Dashboard listing** | ✅ | ✅ |
| **Action tracking** | ✅ | ❌ |
| **Cron integration** | ✅ | ❌ |
| **Own workspace** | ✅ | ❌ |
| **Session isolation** | ✅ | ❌ |
| **Recommended for** | All agents that do work | Inventory/discovery only |

**Rule of thumb:** If the agent will ever run autonomously, use Path A. Always.

---

## 7. Proper Agent Creation — Path A

### Step 1: Create Agent Workspace

```bash
AGENT_NAME="<domain>-<role>"
mkdir -p ~/.openclaw/workspace/agents/${AGENT_NAME}
```

### Step 2: Create Required Files

Each agent folder **MUST** contain:

| File | Purpose | Template |
|------|---------|----------|
| `SOUL.md` | Agent identity, mission, Breadcrumb directive, rules | [See Section 11](#soulmd-template) |
| `USER.md` | Operator context | [See Section 11](#usermd-template) |

Optional but recommended:

| File | Purpose |
|------|---------|
| `AGENT_CONFIG.md` | Structured configuration (not `CONFIG.md` — see [Section 13](#13-agent_configmd-template)) |
| `MEMORY.md` | Persistent state across runs |
| `run.sh` | Fallback bash execution script |

### Step 3: Register as OpenClaw Agent

```bash
source ~/.openclaw/.env 2>/dev/null
openclaw agents add ${AGENT_NAME} \
  --workspace ~/.openclaw/workspace/agents/${AGENT_NAME} \
  --model ${DEFAULT_MODEL:?} \
  --non-interactive
```

This creates:
- OpenClaw agent config entry in `~/.openclaw/openclaw.json`
- Session directory at `~/.openclaw/agents/${AGENT_NAME}/`
- Database record in the telemetry backend

### Step 4: Verify Registration

```bash
# Confirm agent exists in the telemetry backend
curl -s "${OPENCLAW_TELEMETRY_API}/api/agents" | jq '.[] | select(.name=="'"${AGENT_NAME}"'")'

# Confirm agent exists in OpenClaw
openclaw agents list | grep "${AGENT_NAME}"
```

**Expected output:** A JSON object with `id`, `name`, `status`, and `actionCount` fields.

If empty: re-run Step 3. If still empty: check telemetry API logs.

### Step 5: Capture Agent ID

```bash
AGENT_ID=$(curl -s "${OPENCLAW_TELEMETRY_API}/api/agents" | jq -r '.[] | select(.name=="'"${AGENT_NAME}"'") | .id')
echo "Agent ID: ${AGENT_ID}"
```

### Step 6: Create Cron Job

See [Section 10: Cron Setup](#10-cron-setup).

### Step 7: Test Execution

```bash
# Trigger a single manual run
openclaw cron run <cron-id>

# Verify action was recorded (check API or fallback log)
curl -s "${OPENCLAW_TELEMETRY_API}/api/actions" | jq '.[] | select(.agentId=="'"${AGENT_ID}"'") | {actionType, description, createdAt}' | head -20

# If API was unreachable, check fallback log
cat ~/.openclaw/logs/telemetry_fallback.log 2>/dev/null | tail -5
```

---

## 8. Bulk Sync — Path B

### When to Use

- Quickly populate the dashboard from existing agent folders
- Prototyping or evaluating agent concepts before committing to full creation
- Generating an inventory of all defined agents

### How to Execute

```bash
source ~/.openclaw/.env 2>/dev/null
curl -X POST "${OPENCLAW_TELEMETRY_API:?}/api/agents?action=sync"
```

### Limitations

Bulk Sync agents are **read-only dashboard entries**. They cannot:
- Be targeted by cron
- Record actions
- Maintain isolated sessions
- Track execution history

### Promoting to Path A

When a Bulk Sync agent needs to become operational:

```bash
AGENT_NAME="<domain>-<role>"
source ~/.openclaw/.env 2>/dev/null

# 1. Ensure workspace and required files exist
ls ~/.openclaw/workspace/agents/${AGENT_NAME}/SOUL.md

# 2. Register properly
openclaw agents add ${AGENT_NAME} \
  --workspace ~/.openclaw/workspace/agents/${AGENT_NAME} \
  --model ${DEFAULT_MODEL:?} \
  --non-interactive

# 3. Verify promotion
curl -s "${OPENCLAW_TELEMETRY_API}/api/agents" | jq '.[] | select(.name=="'"${AGENT_NAME}"'")'
```

---

## 9. Action Tracking (Async Telemetry)

### Why Async Matters

If the telemetry API hangs, a synchronous `curl` blocks the agent's main thread, causing cron overlaps and session locking. The script below fires in the background, retries on failure, and falls back to a local log if the API is unreachable. **The agent never waits for the database.**

### Action Schema

```json
{
  "agentId": "<uuid>",
  "actionType": "<string>",
  "description": "<string>",
  "status": "success | failure | partial"
}
```

### Standard Action Types

| Action Type | Use When |
|-------------|----------|
| `scan` | Reading/checking a data source |
| `mine` | Extracting structured data |
| `bake` | Discovering, validating, and indexing components |
| `build` | Producing an artifact |
| `compact` | Reducing, cleaning, or archiving |
| `check` | Validating health or integrity |
| `dispatch` | Routing a task to another agent |
| `review` | Evaluating or auditing content |
| `noop` | Agent ran but found nothing to do |

### The Standard Action Script

Create once at `~/.openclaw/scripts/record-action.sh`:

```bash
cat > ~/.openclaw/scripts/record-action.sh << 'SCRIPT'
#!/bin/bash
# record-action.sh — Async, non-blocking telemetry wrapper.
# Fires in the background. Agent never blocks on DB writes.
# Falls back to local log if API is unreachable.

AGENT_NAME="$1"
ACTION_TYPE="$2"
DESCRIPTION="$3"
STATUS="${4:-success}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
FALLBACK_LOG="${HOME}/.openclaw/logs/telemetry_fallback.log"

# ── All heavy lifting runs in a background subshell ──
(
  set -euo pipefail

  # Load environment (suppress errors if .env is missing)
  source ~/.openclaw/.env 2>/dev/null
  API_URL="${OPENCLAW_TELEMETRY_API:-}"

  # If no API configured, log locally and exit
  if [ -z "$API_URL" ]; then
    mkdir -p "$(dirname "$FALLBACK_LOG")"
    echo "[$TIMESTAMP] LOCAL_ONLY | $AGENT_NAME | $ACTION_TYPE | $STATUS | $DESCRIPTION" >> "$FALLBACK_LOG"
    exit 0
  fi

  # Resolve agent name to ID (synchronous — fast lookup with timeout)
  AGENT_ID=$(curl -sf --max-time 5 "$API_URL/api/agents" \
    | jq -r ".[] | select(.name | ascii_downcase == \"$(echo "$AGENT_NAME" | tr '[:upper:]' '[:lower:]')\") | .id" \
    || echo "")

  if [ -z "$AGENT_ID" ] || [ "$AGENT_ID" = "null" ]; then
    mkdir -p "$(dirname "$FALLBACK_LOG")"
    echo "[$TIMESTAMP] AGENT_NOT_FOUND | $AGENT_NAME | $ACTION_TYPE | $STATUS | $DESCRIPTION" >> "$FALLBACK_LOG"
    exit 0
  fi

  # POST with retries and hard timeout
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$API_URL/api/actions" \
    -H "Content-Type: application/json" \
    --max-time 5 \
    --retry 3 \
    --retry-delay 2 \
    --retry-connrefused \
    -d "{\"agentId\": \"$AGENT_ID\", \"actionType\": \"$ACTION_TYPE\", \"description\": \"$DESCRIPTION\", \"status\": \"$STATUS\"}")

  if [[ "$HTTP_CODE" != "200" && "$HTTP_CODE" != "201" ]]; then
    mkdir -p "$(dirname "$FALLBACK_LOG")"
    echo "[$TIMESTAMP] API_FAILED (HTTP $HTTP_CODE) | $AGENT_NAME | $ACTION_TYPE | $STATUS | $DESCRIPTION" >> "$FALLBACK_LOG"
  fi
) &

# Stdout confirmation for cron logs (fires immediately, does not wait for background)
echo "[ACTION] ${AGENT_NAME} | ${ACTION_TYPE} | ${STATUS} | ${TIMESTAMP}"
SCRIPT

chmod +x ~/.openclaw/scripts/record-action.sh
```

### Usage

```bash
# Success
~/.openclaw/scripts/record-action.sh "memory-miner" "mine" "Extracted 12 facts from chat logs"

# Failure
~/.openclaw/scripts/record-action.sh "memory-miner" "mine" "API timeout after 30s" "failure"

# No-op
~/.openclaw/scripts/record-action.sh "feed-scanner" "noop" "No new items since last run"
```

### Fallback Log

When the API is unreachable, actions are written to `~/.openclaw/logs/telemetry_fallback.log` with structured fields:

```
[2026-02-22T14:30:00Z] API_FAILED (HTTP 503) | memory-miner | mine | success | Extracted 12 facts
[2026-02-22T14:30:00Z] AGENT_NOT_FOUND | typo-agent | scan | success | Test run
[2026-02-22T14:30:00Z] LOCAL_ONLY | memory-miner | mine | success | No OPENCLAW_TELEMETRY_API set
```

Review periodically: `cat ~/.openclaw/logs/telemetry_fallback.log`

---

## 10. Cron Setup

### Basic Syntax

```bash
openclaw cron add \
  --name "<Domain> <Role> - <Task>" \
  --every "<duration>" \
  --message "<task instruction>" \
  --agent "<agent-name>" \
  --session "isolated" \
  --timeout-seconds <seconds>
```

### Example: Memory Miner — 6-Hour Scan

```bash
openclaw cron add \
  --name "Memory Miner - Scan & Extract" \
  --every "6h" \
  --message "Read your SOUL.md. Follow the Breadcrumb Directive. Scan recent chat messages for extractable facts. Store findings in MEMORY.md. Record action when complete: ~/.openclaw/scripts/record-action.sh memory-miner mine '<summary>'" \
  --agent "memory-miner" \
  --session "isolated" \
  --timeout-seconds 180
```

### Schedule Options

| Format | Example | Description |
|--------|---------|-------------|
| `--every "6h"` | Every 6 hours | Simple interval |
| `--every "30m"` | Every 30 minutes | High-frequency |
| `--cron "30 2 * * *"` | 02:30 daily | Specific daily time |
| `--cron "0 3 * * 0"` | 03:00 Sunday | Weekly |
| `--cron "0 */4 * * *"` | Every 4 hours on the hour | Fixed interval aligned to clock |

### ⚠️ Schedule Alignment

If this agent interacts with shared files (`MEMORY.md`, `recipe.json`, CRM profiles), you **must** align its schedule with the Integration Bridge's Unified Cron Schedule. Verify that the agent's write window does not overlap with existing miner or baker agents to prevent lock collisions.

```bash
# List all existing cron schedules before adding a new one
openclaw cron list --json | jq '.[] | {name, agent, schedule: (.every // .cron)}'
```

### Session Types

| Session | Use When | Tradeoff |
|---------|----------|----------|
| `isolated` | Agent should not see main conversation | Clean context, no bleed |
| `main` | Agent shares context with main agent | Shared state, risk of pollution |

**Default to `isolated` for all cron jobs.** Use `main` only when the agent explicitly needs access to the primary conversation context.

### Timeout Guidance

| Agent Role | Suggested Timeout | Rationale |
|------------|-------------------|-----------|
| Scanner | 60–120s | Quick read operations |
| Miner | 120–300s | Processing + extraction |
| Baker | 120–180s | Discovery + indexing |
| Builder | 300–600s | Report generation |
| Compactor | 180–300s | Data reduction |
| Checker / Auditor | 30–60s | Health pings |
| Keeper / Manager | 180–300s | Identity evolution |

---

## 11. Agent Folder Structure

```
~/.openclaw/workspace/agents/
└── <agent-name>/
    ├── SOUL.md                  # REQUIRED — Agent identity & instructions
    ├── USER.md                  # REQUIRED — Operator context
    ├── AGENT_CONFIG.md          # Recommended — Structured config (NOT CONFIG.md)
    ├── MEMORY.md                # Optional — Persistent state
    ├── run.sh                   # Optional — Fallback bash script
    ├── logs/                    # Auto-created — Execution logs
    │   └── YYYY-MM-DD.log
    └── agent.<name>.crumb.md   # Optional — Auto-indexed by Breadcrumb Baker
```

> **Why `AGENT_CONFIG.md` instead of `CONFIG.md`?** The Lab Manager's deployment auditor parses `CONFIG.md` at the system level. Using the same filename inside agent folders creates parsing collisions.

### SOUL.md Template

```markdown
# <Agent Name>

## Role
<One-line role description based on taxonomy (e.g., "Memory domain miner agent")>

## Mission
<2-3 sentences: what this agent does and why it exists>

## Schedule
<Cron expression or interval — e.g., "Every 6 hours via openclaw cron">
*Note: Verify schedule aligns with the Integration Bridge to prevent lock collisions.*

## Breadcrumb Directive (MANDATORY)
BEFORE executing any task that involves external services, APIs, tools, or any
component you did not build in THIS session:
1. Read `~/breadcrumb-trail/recipe.json`
2. Identify which capabilities are needed for the task
3. For each needed capability, read the `.crumb.md` file at the path listed in `recipe.json`

## Execution Steps
1. Read this SOUL.md to confirm identity and instructions
2. Follow the Breadcrumb Directive above
3. <Primary task step>
4. <Secondary task step>
5. Record action: `~/.openclaw/scripts/record-action.sh "<name>" "<type>" "<description>"`

## Output
- <What this agent produces or modifies>

## Rules
1. Always record an action on completion — even if the action is "noop"
2. Never modify files outside your workspace unless explicitly instructed
3. Search `~/breadcrumb-trail/index.json` before asking the operator how a component works
4. <Agent-specific rule>

## Error Handling
- If primary task fails: record action with status "failure" and error detail
- If API unreachable: rely on `record-action.sh` background retries. Do not hang the session.
```

### USER.md Template

```markdown
# User Context for <Agent Name>

## Parent Context
- Main USER.md: `~/.openclaw/USER.md`
- This agent operates within the main OpenClaw agent ecosystem

## Agent-Specific Context
- Runs as: isolated agent (no access to main conversation)
- Telemetry: via `$OPENCLAW_TELEMETRY_API` (set in `~/.openclaw/.env`)
- Access to: <list systems/APIs this agent may call>

## Operator Notes
- <Any special instructions or constraints>
```

---

## 12. File Manifest

| File | Required | Purpose | Created By |
|------|----------|---------|------------|
| `SOUL.md` | ✅ | Agent identity, Breadcrumb directive, instructions, rules | Operator |
| `USER.md` | ✅ | Operator context and permissions | Operator |
| `AGENT_CONFIG.md` | Recommended | Structured configuration (avoids `CONFIG.md` collision) | Operator |
| `MEMORY.md` | Optional | Persistent state across runs | Agent (auto) |
| `run.sh` | Optional | Fallback bash execution | Operator |
| `logs/` | Auto | Timestamped execution logs | System |
| `CHANGELOG.md` | Optional | Version and change history | Operator |
| `agent.<name>.crumb.md` | Optional | Auto-indexed by Breadcrumb Baker | Operator |

---

## 13. AGENT_CONFIG.md Template

> **Why this filename?** The master system `CONFIG.md` is parsed by the Lab Manager's deployment auditor. Using `AGENT_CONFIG.md` prevents collisions.

```markdown
# Agent Configuration: <agent-name>

## Identity
- **Agent Name:** `<agent-name>`
- **Agent Type:** `<role from taxonomy>`
- **Model:** `${DEFAULT_MODEL}` (set in `~/.openclaw/.env`)

## Scheduling
- **Schedule:** `<cron expression or interval>`
- **Timeout:** `180` seconds
- **Aligned With:** <Integration Bridge schedule reference, if applicable>

## Action Tracking
- **Action Type:** `<scan|mine|bake|build|compact|check|dispatch|review|noop>`
- **Description Template:** `<what to record on each run>`

## Resources
- **Workspace:** `~/.openclaw/workspace/agents/<agent-name>`
- **Telemetry:** `$OPENCLAW_TELEMETRY_API` (set in `~/.openclaw/.env`)

## Dependencies
- **Depends On:** <list of agents or services this agent requires>
- **Breadcrumb Capabilities:** <list of recipe.json capabilities used>
- **Files Read:** <paths this agent reads from>
- **Files Written:** <paths this agent writes to>
```

---

## 14. Agent Lifecycle Management

### Lifecycle States

```
DRAFT ──▶ REGISTERED ──▶ SCHEDULED ──▶ ACTIVE ──▶ DISABLED ──▶ DECOMMISSIONED
  │                                       │            │
  │                                       └── ERRORED ─┘
  └── (abandoned, never registered)
```

| State | Meaning | Dashboard Visible |
|-------|---------|-------------------|
| **Draft** | Folder exists, not registered | ❌ (unless bulk synced) |
| **Registered** | In telemetry DB, no cron | ✅ — 0 actions |
| **Scheduled** | Cron created, not yet run | ✅ — 0 actions |
| **Active** | Running on schedule, recording actions | ✅ — action count > 0 |
| **Errored** | Running but failing (actions with `failure` status) | ✅ — check action status |
| **Disabled** | Cron paused, agent still registered | ✅ — stale action count |
| **Decommissioned** | Removed from cron + DB | ❌ |

### Disabling an Agent

```bash
# Remove cron (agent stays in DB for historical records)
openclaw cron list | grep "<agent-name>"
openclaw cron remove <cron-id>

# Optional: mark as disabled in AGENT_CONFIG.md
echo "- **Status:** DISABLED ($(date -I))" >> ~/.openclaw/workspace/agents/<agent-name>/AGENT_CONFIG.md
```

### Decommissioning an Agent

```bash
AGENT_NAME="<domain>-<role>"

# 1. Remove cron job
openclaw cron remove <cron-id>

# 2. Archive workspace (preserve history)
mkdir -p ~/.openclaw/archive
tar -czf ~/.openclaw/archive/${AGENT_NAME}-$(date +%Y%m%d).tar.gz \
  ~/.openclaw/workspace/agents/${AGENT_NAME}/

# 3. Remove workspace
rm -rf ~/.openclaw/workspace/agents/${AGENT_NAME}

# 4. Remove from OpenClaw
openclaw agents remove ${AGENT_NAME} 2>/dev/null || echo "Manual DB cleanup may be required"
```

---

## 15. Health Checks & Monitoring

### Stale Agent Detection

An agent is **stale** if its last action is older than 2× its cron interval.

```bash
#!/bin/bash
# ~/.openclaw/scripts/check-agent-health.sh
source ~/.openclaw/.env 2>/dev/null
API_URL="${OPENCLAW_TELEMETRY_API:?OPENCLAW_TELEMETRY_API environment variable is not set}"
STALE_THRESHOLD_HOURS="${1:-24}"

echo "=== Agent Health Check (stale threshold: ${STALE_THRESHOLD_HOURS}h) ==="

AGENTS=$(curl -sf "$API_URL/api/agents" | jq -c '.[]')

while IFS= read -r agent; do
  NAME=$(echo "$agent" | jq -r '.name')
  ACTION_COUNT=$(echo "$agent" | jq -r '.actionCount // 0')
  LAST_ACTION=$(echo "$agent" | jq -r '.lastActionAt // "never"')

  if [ "$ACTION_COUNT" -eq 0 ]; then
    echo "[WARN]  ${NAME}: 0 actions (zombie agent)"
  elif [ "$LAST_ACTION" != "never" ]; then
    # NOTE: `date -d` is GNU coreutils. macOS users need `gdate` from `brew install coreutils`.
    LAST_EPOCH=$(date -d "$LAST_ACTION" +%s 2>/dev/null || gdate -d "$LAST_ACTION" +%s 2>/dev/null || echo 0)
    NOW_EPOCH=$(date +%s)
    HOURS_AGO=$(( (NOW_EPOCH - LAST_EPOCH) / 3600 ))
    if [ "$HOURS_AGO" -gt "$STALE_THRESHOLD_HOURS" ]; then
      echo "[STALE] ${NAME}: last action ${HOURS_AGO}h ago"
    else
      echo "[OK]    ${NAME}: last action ${HOURS_AGO}h ago (${ACTION_COUNT} total)"
    fi
  fi
done <<< "$AGENTS"
```

### Health Check as an Agent

```bash
openclaw agents add health-checker \
  --workspace ~/.openclaw/workspace/agents/health-checker \
  --model ${DEFAULT_MODEL:?} \
  --non-interactive

openclaw cron add \
  --name "Health Checker - Agent Status" \
  --every "2h" \
  --message "Run ~/.openclaw/scripts/check-agent-health.sh. Report any WARN or STALE agents. Record action with findings summary." \
  --agent "health-checker" \
  --session "isolated" \
  --timeout-seconds 60
```

### Key Metrics to Monitor

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Action count trend | Increasing | Flat for >24h | Decreasing or 0 |
| Last action age | < 1× cron interval | 1–2× interval | > 2× interval |
| Failure ratio | < 5% | 5–20% | > 20% |
| Cron execution | On schedule | Occasional skip | Multiple consecutive misses |
| Fallback log size | Empty or minimal | Growing | Growing rapidly |

---

## 16. Security Considerations

### Network Isolation

- The telemetry API should listen on `127.0.0.1` only (not `0.0.0.0`) unless explicitly needed by remote agents
- All inter-agent communication should traverse `localhost` or a trusted internal network
- If exposing the telemetry API externally, use a reverse proxy with authentication
- External APIs should be verified via the Breadcrumb `recipe.json` capabilities list before agents execute calls against them

### API Authentication

- The default telemetry API is unauthenticated — acceptable for single-operator localhost deployments
- For multi-user or remote access: add API key validation via middleware or reverse proxy
- Rotate any API keys if agent credentials are exposed

### Agent Permissions

- Agents should only have filesystem access to their own workspace (plus `~/breadcrumb-trail/` read-only)
- `SOUL.md` should explicitly declare which APIs and paths the agent may access
- Avoid granting agents write access to other agents' workspaces

### Secrets Management

- Never embed API keys, tokens, or IPs in `SOUL.md`, `AGENT_CONFIG.md`, or cron messages
- Use `~/.openclaw/.env` for all system-level variables
- Agent scripts must explicitly `source ~/.openclaw/.env` to retrieve credentials
- All scripts use `${VAR:?}` syntax — they refuse to run if required variables are unset

---

## 17. Multi-Agent Coordination

### Patterns

**Sequential Pipeline:** Agent A outputs data → Agent B consumes it.

```
feed-scanner ──writes──▶ ~/.openclaw/shared/feeds/raw.json
                              │
memory-miner ──reads───▶ ~/.openclaw/shared/feeds/raw.json
             ──writes──▶ ~/.openclaw/shared/feeds/processed.json
```

**Shared State via Files:**
- Use `~/.openclaw/shared/` as the inter-agent data directory
- Each agent reads from shared, writes to its own output path
- Naming: `<agent-name>-output.json` or `<agent-name>-<date>.json`

**Dispatcher Pattern:** A dispatcher agent reads a task queue and triggers specialist agents.

```bash
# Dispatcher cron message example:
"Check ~/.openclaw/shared/task-queue.json for pending tasks. 
 For each task, determine the appropriate specialist agent and 
 record a dispatch action. Write task assignments to 
 ~/.openclaw/shared/dispatch-log.json."
```

### Coordination Rules

1. **No direct agent-to-agent calls** — agents communicate through shared files in `~/.openclaw/shared/` or the Telemetry API
2. **Timestamp all shared files** — include `updatedAt` in any shared JSON
3. **Idempotent reads** — an agent reading shared state must handle stale or missing data gracefully
4. **One writer per file** — if multiple agents need to write, use separate output files to prevent locks
5. **Verify schedule alignment** — agents sharing files must not have overlapping write windows (see [Section 10: Schedule Alignment](#⚠️-schedule-alignment))

---

## 18. Troubleshooting

### Agent Shows 0 Actions

| Possible Cause | Diagnostic | Fix |
|----------------|------------|-----|
| `record-action.sh` not called in cron message | Review cron `--message` | Add async action recorder call |
| Agent ID mismatch | Compare cron agentId to DB | Re-capture ID from API |
| Telemetry API unreachable | Check `~/.openclaw/logs/telemetry_fallback.log` | Verify `.env` variables and API health |
| Agent was bulk-synced (Path B) | Check if cron exists | Promote to Path A |
| `.env` not sourced in script | `echo $OPENCLAW_TELEMETRY_API` returns empty | Add `source ~/.openclaw/.env` |

### Cron Never Triggers

```bash
# 1. Verify agent exists in OpenClaw
openclaw agents list | grep "<agent-name>"

# 2. Verify cron exists and is enabled
openclaw cron list | grep "<agent-name>"

# 3. Check cron schedule format
openclaw cron list --json | jq '.[] | select(.agent=="<agent-name>")'

# 4. Trigger manually to isolate cron vs agent issues
openclaw cron run <cron-id>
```

### Agent Runs But No DB Entry

**Root cause:** Agent not created via Path A.

```bash
source ~/.openclaw/.env 2>/dev/null
openclaw agents add <agent-name> \
  --workspace ~/.openclaw/workspace/agents/<agent-name> \
  --model ${DEFAULT_MODEL:?} \
  --non-interactive
```

### Sub-Agent Spawn Fails

**Root cause:** Gateway pairing issue.

```bash
openclaw devices list              # Check paired devices
openclaw devices clear --yes       # Clear if needed
# Re-pair via control UI
```

### Agent Runs But Produces Wrong Output

| Possible Cause | Diagnostic | Fix |
|----------------|------------|-----|
| Stale `SOUL.md` | Review file contents | Update instructions |
| Missing Breadcrumb Directive | Check SOUL.md for `recipe.json` step | Add directive from template |
| Wrong model specified | Check `openclaw.json` | Re-add with correct `--model` |
| Context pollution from `main` session | Check `--session` type | Switch to `isolated` |
| Workspace path mismatch | Compare `--workspace` to actual path | Re-add with correct path |

### Action Recording Fails Silently

```bash
# Check the fallback log first — this catches all async failures
cat ~/.openclaw/logs/telemetry_fallback.log 2>/dev/null | tail -10

# Test the script directly
source ~/.openclaw/.env 2>/dev/null
~/.openclaw/scripts/record-action.sh "<agent-name>" "check" "test action"

# Common issues:
# - OPENCLAW_TELEMETRY_API not set (script will error immediately)
# - Agent name case mismatch
# - jq not installed
which jq || echo "jq is required but not installed"
```

---

## 19. Build Order

### New Agent Checklist

```
[ ] 1.  Choose name per taxonomy (<domain>-<role>)
[ ] 2.  Create workspace:          mkdir -p ~/.openclaw/workspace/agents/<name>
[ ] 3.  Write SOUL.md:             Identity, Breadcrumb Directive, async action tracker
[ ] 4.  Write USER.md:             Operator context and permissions
[ ] 5.  Write AGENT_CONFIG.md:     Schedule, dependencies (no hardcoded IPs)
[ ] 6.  Source environment:         source ~/.openclaw/.env
[ ] 7.  Register agent:            openclaw agents add <name> --workspace <path> --model ${DEFAULT_MODEL} --non-interactive
[ ] 8.  Verify registration:       curl + jq to confirm DB entry
[ ] 9.  Capture agent ID:          Store for reference
[ ] 10. Check schedule alignment:  openclaw cron list --json (verify no overlaps)
[ ] 11. Create cron:               openclaw cron add --agent <name> --session isolated
[ ] 12. Test manually:             openclaw cron run <cron-id>
[ ] 13. Verify action:             Check DB or fallback log for async action post
[ ] 14. Monitor first 24h:         Confirm scheduled runs are executing
```

---

## 20. Quick Reference

```bash
# ── Source Environment (always do this first) ──
source ~/.openclaw/.env

# ── Agent Creation ──
openclaw agents add <name> \
  --workspace ~/.openclaw/workspace/agents/<name> \
  --model ${DEFAULT_MODEL} \
  --non-interactive

# ── List Agents ──
openclaw agents list
curl -s "${OPENCLAW_TELEMETRY_API}/api/agents" | jq '.[].name'

# ── Create Cron ──
openclaw cron add \
  --name "<Domain> <Role> - <Task>" \
  --every "<duration>" \
  --message "<task instruction>" \
  --agent "<name>" \
  --session isolated \
  --timeout-seconds <seconds>

# ── Record Action (Async) ──
~/.openclaw/scripts/record-action.sh "<name>" "<type>" "<description>"

# ── Check Agent Status ──
curl -s "${OPENCLAW_TELEMETRY_API}/api/agents" | jq '.[] | select(.name=="<name>")'

# ── Check Fallback Log ──
cat ~/.openclaw/logs/telemetry_fallback.log

# ── Health Check ──
~/.openclaw/scripts/check-agent-health.sh 24

# ── Bulk Sync (Path B) ──
curl -X POST "${OPENCLAW_TELEMETRY_API}/api/agents?action=sync"

# ── Decommission ──
openclaw cron remove <cron-id>
tar -czf ~/.openclaw/archive/<name>-$(date +%Y%m%d).tar.gz ~/.openclaw/workspace/agents/<name>/
rm -rf ~/.openclaw/workspace/agents/<name>
openclaw agents remove <name>
```

---

## 21. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 3.0 | 2026-02-22 | **Ecosystem integration:** Added mandatory Breadcrumb Directive to SOUL.md template. Integrated `recipe.json` lookup into execution flow and architecture diagram. Added `bake` action type for Baker agents. **Naming:** Shifted taxonomy from `<function>-<qualifier>` to `<domain>-<role>` to align with existing agent roster. Added Baker, Auditor, Keeper, Manager archetypes. **Config collision fix:** Renamed `CONFIG.md` → `AGENT_CONFIG.md` to prevent Lab Manager deployment auditor conflicts. **Telemetry hardening:** Rebuilt `record-action.sh` with `set -euo pipefail` inside background subshell, `--retry 3 --retry-delay 2 --retry-connrefused` on POST, local fallback log at `~/.openclaw/logs/telemetry_fallback.log` with structured fields, HTTP status code validation. **Schedule safety:** Added schedule alignment warning and `openclaw cron list` verification step for shared-file agents. Added "Aligned With" field to AGENT_CONFIG.md. **Principles:** Added "Agnostic observability", "Non-blocking telemetry", "Ecosystem alignment". **Build order:** Added `source .env` step and schedule alignment check. |
| 2.1 | 2025-02-22 | Architecture fixes: async `curl` via background subshell, removed `exit 1` risk. Hardcoding eliminated: `$OPENCLAW_TELEMETRY_API` and `$DEFAULT_MODEL` env vars. |
| 2.0 | 2025-02-22 | Lifecycle management, taxonomy introduction, health checks, security, multi-agent coordination, troubleshooting matrix. |
| 1.0 | — | Initial release. Core agent creation, action tracking, cron setup, folder structure. |
