# OpenClaw Integration Bridge — Tri-Domain (Memory × Breadcrumbs × CRM)

*Version: 2.1 | Spec ID: `openclaw-integration-bridge-v2.1`*
*Audience: OpenClaw operators running two or more primary domain specs.*
*License: Share freely.*

---

## Changelog

| Version | Date | Changes |
|:--------|:-----|:--------|
| **2.1** | 2026-02-22 | Added **Cron Sentinels** (§2.1) for overlap prevention and overrun detection. Added **CONFIG Override Rule** (§3.1) to enforce single-source-of-truth budgets. Added **baker-reindex.sh specification** (§5.1) with argument contract and REINDEX_RESULT format. |
| **2.0** | — | Initial tri-domain bridge: unified cron schedule, modular CONFIG, context assembler, recall routing, CRM exclusion protocol, error recovery. |

---

## 1. Purpose

The Persistent Memory, Breadcrumb System, and CRM System specs are designed to function independently. This Bridge provides the coordination required to prevent data race conditions, context window overflow, and "identity confusion" when these systems share the same OpenClaw instance.

### Modular "Fail-Soft" Design

* **If CRM is missing**: The Bridge ignores the `crm_tokens` and `crm_entity_exclusion_source` directives; the Context Assembler reallocates that budget to Long-Term Memory automatically.
* **If Breadcrumbs are missing**: The `external_context` slots remain empty, and the system functions as a standard Memory/CRM setup.

---

## 2. Unified Cron Schedule

This schedule ensures that write-intensive tasks (like indexing and mining) do not overlap, preventing filesystem locks.

| Time | Process | System | Phase / Notes |
|:---|:---|:---|:---|
| **02:30** | Soul Keeper | Memory | Identity evolution and SOUL.md updates. |
| **05:00** | Breadcrumb Baker | Breadcrumb | Discover + Validate + Index tools/capabilities. |
| **06:00** | Memory Miner | Memory | Extract chronological memories from the previous day. |
| **06:30** | CRM Miner | CRM | **(Optional)** Update relationship/entity profiles. |
| **07:00** | Conflict Detector | Memory | Cross-source consistency check across all logs. |
| **11:00** | Breadcrumb Baker | Breadcrumb | Standard mid-day tool reindex. |
| **17:00** | Breadcrumb Baker | Breadcrumb | Standard afternoon tool reindex. |
| **23:00** | Breadcrumb Baker | Breadcrumb | Final daily tool reindex. |

> **Note:** All scheduled jobs in this table MUST use the Cron Sentinel system (§2.1) to guard against overlap and detect overruns.

### 2.1 Cron Sentinels

Cron jobs that run on a schedule can overlap if a previous run has not completed. The **Cron Sentinel** system prevents this by using lightweight sentinel files as coordination locks.

#### How It Works

1. **On Start:** When a cron job begins, it creates a sentinel file in the `cron_sentinel_directory`. The file is named after the job (e.g., `breadcrumb-baker.sentinel`) and contains the PID and start timestamp.
2. **On Completion:** When the job finishes (success or handled failure), it removes its sentinel file.
3. **On Next Run:** Before starting, the job checks for an existing sentinel:
   - **Sentinel exists AND age < `cron_overrun_wait_minutes`:** The previous run is still active. **Skip this run** and log a warning.
   - **Sentinel exists AND age ≥ `cron_overrun_wait_minutes`:** The previous run likely crashed (stale sentinel). **Remove the stale sentinel and proceed** with the new run. Log the stale sentinel detection for diagnostics.
   - **No sentinel exists:** Proceed normally.

#### CONFIG Parameters

```yaml
# Cron Sentinel Configuration
cron_sentinel_directory: "~/.openclaw/workspace/cron-sentinels/"
cron_overrun_wait_minutes: 10
```

| Parameter | Default | Description |
|:---|:---|:---|
| `cron_sentinel_directory` | `~/.openclaw/workspace/cron-sentinels/` | Directory where sentinel files are created. Must be writable. Created automatically on first use. |
| `cron_overrun_wait_minutes` | `10` | Age threshold (in minutes) for sentinel staleness detection. If a sentinel file is older than this value, the previous run is assumed to have crashed. |

#### Sentinel File Format

Each sentinel file is a plain text file with the following content:

```
PID=<process_id>
STARTED=<ISO-8601 timestamp>
JOB=<job_name>
```

**Example** (`~/.openclaw/workspace/cron-sentinels/breadcrumb-baker.sentinel`):

```
PID=48291
STARTED=2026-02-22T05:00:01-05:00
JOB=breadcrumb-baker
```

#### Reference Implementation (Shell)

```bash
#!/usr/bin/env bash
# sentinel-guard.sh — Source this in any cron job for automatic sentinel protection.
# Usage: source sentinel-guard.sh "job-name"

JOB_NAME="${1:?Usage: source sentinel-guard.sh <job-name>}"
SENTINEL_DIR="${CRON_SENTINEL_DIRECTORY:-$HOME/.openclaw/workspace/cron-sentinels}"
OVERRUN_MINUTES="${CRON_OVERRUN_WAIT_MINUTES:-10}"
SENTINEL_FILE="${SENTINEL_DIR}/${JOB_NAME}.sentinel"

mkdir -p "$SENTINEL_DIR"

if [ -f "$SENTINEL_FILE" ]; then
  FILE_AGE_SEC=$(( $(date +%s) - $(stat -f %m "$SENTINEL_FILE" 2>/dev/null || stat -c %Y "$SENTINEL_FILE") ))
  THRESHOLD_SEC=$(( OVERRUN_MINUTES * 60 ))
  if [ "$FILE_AGE_SEC" -lt "$THRESHOLD_SEC" ]; then
    echo "[SENTINEL] $JOB_NAME: previous run still active (age ${FILE_AGE_SEC}s < ${THRESHOLD_SEC}s). Skipping."
    exit 0
  else
    echo "[SENTINEL] $JOB_NAME: stale sentinel detected (age ${FILE_AGE_SEC}s >= ${THRESHOLD_SEC}s). Previous run likely crashed. Proceeding."
    rm -f "$SENTINEL_FILE"
  fi
fi

# Create sentinel
cat > "$SENTINEL_FILE" <<EOF
PID=$$
STARTED=$(date -Iseconds)
JOB=$JOB_NAME
EOF

# Ensure sentinel is removed on exit (success, failure, or signal)
trap "rm -f '$SENTINEL_FILE'" EXIT INT TERM
```

---

## 3. Modular CONFIG.md Architecture

Add these values to your central `CONFIG.md`. The logic is designed to be "null-safe"—if a path does not exist, the specific integration hook remains inert.

```yaml
# ══════════════════════════════════════════════════════════════
# Integration Bridge v2.1 - Token Budgets & Routing
# ══════════════════════════════════════════════════════════════
context_budget_tokens: 6000

# Primary Domain Budgets
identity_tokens: 800
user_profile_tokens: 600
corrections_tokens: 400
crm_tokens: 1000                # Set to 0 if CRM spec is not used
external_context_budget: 800    # Reserved for Breadcrumbs

# Secondary/Recall Budgets
recent_memory_tokens: 1100
semantic_recall_tokens: 1000
longterm_memory_tokens: 300     # Absorbs CRM/External budget if null

# Integration Hooks
external_context_source: "~/breadcrumb-trail/recipe.json"
recall_external_handler: "~/breadcrumb-trail/scripts/baker-reindex.sh"
crm_entity_exclusion_source: "~/breadcrumb-trail/recipe.json"

# Cron Sentinel Configuration (v2.1)
cron_sentinel_directory: "~/.openclaw/workspace/cron-sentinels/"
cron_overrun_wait_minutes: 10

heartbeat_custom_patterns:
  - name: "breadcrumb_triggers"
    patterns:
      - '/broke|broken|stopped working|not working/i'
      - '/(there.s|we have|we built|we made|i made) (already |)(a |an |)(script|tool|workflow|bot|agent|dashboard|pipeline)/i'
      - '/build me a/i'
      - '/what (tools|scripts|capabilities|integrations) do (we|you|i) have/i'
    route_to: "breadcrumb-directive"
    priority: "normal"
```

### 3.1 CONFIG Override Rule — Single Source of Truth for Token Budgets

When the Integration Bridge CONFIG.md defines token budgets, it becomes the **single authoritative source** for all budget values. This prevents conflicting or duplicated budget definitions across specs.

#### Rules

1. **Bridge budgets win.** If a token budget key (e.g., `identity_tokens`, `recent_memory_tokens`) is defined in the Integration Bridge CONFIG.md, that value is canonical. Individual spec CONFIG.md files MUST NOT define the same key with a different value.

2. **Comment out or remove duplicates.** Any individual domain spec (Memory, Breadcrumb, CRM) that previously defined its own token budgets MUST have those values either:
   - **Commented out** with a note pointing to the Bridge, e.g.:
     ```yaml
     # identity_tokens: 800  # MOVED TO: Integration Bridge CONFIG (§3)
     ```
   - **Removed entirely**, with a header note:
     ```yaml
     # Token budgets managed by Integration Bridge v2.1 — see §3
     ```

3. **Budget validation constraint.** The sum of all individual slot budgets MUST NOT exceed `context_budget_tokens`:

   ```
   identity_tokens + user_profile_tokens + corrections_tokens + crm_tokens
   + external_context_budget + recent_memory_tokens + semantic_recall_tokens
   + longterm_memory_tokens  ≤  context_budget_tokens
   ```

   With the default values:
   ```
   800 + 600 + 400 + 1000 + 800 + 1100 + 1000 + 300 = 6000 ≤ 6000 ✓
   ```

4. **Deployment Auditor enforcement.** The Deployment Auditor SHOULD verify this constraint during commissioning. If the sum exceeds the budget, the audit MUST fail with a clear error identifying which slots overflow.

---

## 4. Context Assembler — Loading Priority

When the agent initializes, the Context Assembler loads data in this specific order to ensure the most "stable" facts (Identity and Tools) are never truncated by "volatile" facts (Recent Memories):

1. **Identity (`SOUL.md`)**: Who the agent is.
2. **Operator Baseline (`USER.md`)**: Who they are talking to.
3. **Capability Map (`recipe.json`)**: What tools are available (Breadcrumb summary).
4. **CRM Profiles**: Who/What is involved in the current task (if CRM exists).
5. **Recent Corrections**: Immediate feedback from the operator.
6. **Recent Memory**: The most immediate conversational context.
7. **Semantic Recall**: Relevant past experiences.
8. **Long-Term Memory**: Deep historical context.

---

## 5. Recall Failure Routing

The Recall Validator uses a three-way branch to handle corrections. If an agent "forgets" something, it classifies the failure to ensure the right database is updated:

* **Type: Tool / Capability** → Route to `baker-reindex.sh` (see §5.1). This triggers Baker to see if a tool is documented but missing from the index.
* **Type: Relationship / Entity** → Route to the CRM Archive. Updates the specific Markdown profile for that person or place.
* **Type: Chronological / Knowledge** → Route to the corrections log and `MEMORY.md`. Standard chronological memory update.

### 5.1 baker-reindex.sh — Breadcrumb Reindex Script

The `baker-reindex.sh` script is the Bridge's primary hook into the Breadcrumb Baker for targeted re-indexing. It is invoked by the Recall Failure Router (§5) and can also be called directly when new components are added to the system.

#### Location

```
~/breadcrumb-trail/scripts/baker-reindex.sh
```

The script MUST be executable (`chmod +x`).

#### Arguments

| Position | Name | Required | Description |
|:---------|:-----|:---------|:------------|
| `$1` | `search_scope` | **Yes** | The scope to search. One of: `tools`, `agents`, `workflows`, `all`. Determines which breadcrumb directories are scanned. |
| `$2` | `filter` | No | Optional keyword or glob pattern to narrow the reindex to specific entries (e.g., `"gmail*"`, `"memory-miner"`). If omitted, the full scope is reindexed. |
| `$3+` | Additional filters | No | Additional filter arguments are OR'd together (any match triggers reindex for that entry). |

**Minimum arguments:** 2 (the script MUST accept at least `search_scope`; `filter` is optional but the argument position must be supported).

#### Output Format — REINDEX_RESULT

The script MUST output a structured result block to stdout upon completion:

```
===== REINDEX_RESULT =====
SCOPE=<search_scope>
FILTER=<filter_value or "none">
INDEXED=<count of entries indexed>
SKIPPED=<count of entries skipped (unchanged)>
ERRORS=<count of entries that failed>
DURATION=<elapsed seconds>
TIMESTAMP=<ISO-8601>
===== END_REINDEX_RESULT =====
```

**Example output:**

```
===== REINDEX_RESULT =====
SCOPE=tools
FILTER=gmail*
INDEXED=3
SKIPPED=12
ERRORS=0
DURATION=4
TIMESTAMP=2026-02-22T11:00:07-05:00
===== END_REINDEX_RESULT =====
```

#### Exit Codes

| Code | Meaning |
|:-----|:--------|
| `0` | Success — reindex completed (ERRORS may still be > 0 for partial success). |
| `1` | Fatal error — reindex could not start (missing directory, bad scope, etc.). |
| `2` | Invalid arguments — wrong number or unrecognized scope. |

#### Reference Skeleton

```bash
#!/usr/bin/env bash
# baker-reindex.sh — Targeted reindex for the Breadcrumb Baker
# Location: ~/breadcrumb-trail/scripts/baker-reindex.sh
# Usage: baker-reindex.sh <scope> [filter...]
#   scope: tools | agents | workflows | all
#   filter: optional keyword/glob to narrow reindex

set -euo pipefail

SCOPE="${1:?Usage: baker-reindex.sh <scope> [filter...]}"
shift
FILTERS=("$@")

VALID_SCOPES=("tools" "agents" "workflows" "all")
if [[ ! " ${VALID_SCOPES[*]} " =~ " ${SCOPE} " ]]; then
  echo "ERROR: Invalid scope '$SCOPE'. Must be one of: ${VALID_SCOPES[*]}" >&2
  exit 2
fi

BREADCRUMB_DIR="${BREADCRUMB_TRAIL_DIR:-$HOME/breadcrumb-trail}"
START_TIME=$(date +%s)
INDEXED=0
SKIPPED=0
ERRORS=0
FILTER_DISPLAY="${FILTERS[*]:-none}"

# --- Core reindex logic goes here ---
# Scan $BREADCRUMB_DIR based on $SCOPE
# Apply $FILTERS to narrow results
# Update recipe.json with discovered/changed breadcrumbs
# Increment INDEXED / SKIPPED / ERRORS counters
# ---

END_TIME=$(date +%s)
DURATION=$(( END_TIME - START_TIME ))

cat <<EOF
===== REINDEX_RESULT =====
SCOPE=$SCOPE
FILTER=$FILTER_DISPLAY
INDEXED=$INDEXED
SKIPPED=$SKIPPED
ERRORS=$ERRORS
DURATION=$DURATION
TIMESTAMP=$(date -Iseconds)
===== END_REINDEX_RESULT =====
EOF

exit 0
```

#### Integration Notes

- The Bridge CONFIG's `recall_external_handler` points to this script.
- The Cron Schedule (§2) triggers Baker runs at 05:00, 11:00, 17:00, and 23:00 — each of those runs should invoke `baker-reindex.sh all` internally.
- When a new component is added (new skill, new agent, new workflow), the Bridge may invoke `baker-reindex.sh <scope> <component-name>` immediately rather than waiting for the next scheduled run.
- The script MUST use Cron Sentinels (§2.1) if invoked from a cron job. Ad-hoc invocations (e.g., from recall routing) do not require sentinels.

---

## 6. CRM Entity Exclusion Protocol

To prevent "Profile Bloat," the CRM Miner must check the Breadcrumb system before creating a new entity profile.

### The Logic

1. Parse the operator mention.
2. Check `recipe.json` capability keys and summary strings for matches.
3. **If matched**: Skip full profile creation. Append a cross-reference note into the operator's baseline profile (e.g., *"Uses [tool] — see [breadcrumb_id] for integration details"*).
4. **If NOT matched**: Proceed with standard CRM entity profile creation.

---

## 7. Error Recovery & Unified Fallbacks

If any single system fails, the Bridge provides a "graceful degradation" path handled natively by the Context Assembler:

| Failed Resource | Unified Fallback Sequence | System |
|:---|:---|:---|
| Agent Identity | Latest snapshot from `SOUL_HISTORY/`. | Memory |
| Vector DB | Keyword search against chronological memory and CRM archive. | Memory/CRM |
| Capabilities Manifest | Filesystem scan for `*.crumb.md` extensions. | Breadcrumb |
| CRM Archive | Skip slot; reallocate CRM tokens to Long-Term Memory. | CRM |
| Cron Sentinel Directory | Fall back to `/tmp/openclaw-sentinels/` with a logged warning. | Bridge |

---

## 8. Build Order

1. **Verify Individual Deployments**: Confirm Memory, Breadcrumbs, and (optionally) CRM are fully operational in isolation.
2. **Apply Configuration**: Update your master `CONFIG.md` with the integrated token budgets and routing parameters from Section 3. **Remove or comment out** duplicate budget definitions in individual spec CONFIG files per the Override Rule (§3.1).
3. **Validate Budget Constraint**: Confirm the sum of all slot budgets ≤ `context_budget_tokens` (§3.1).
4. **Create Sentinel Directory**: Ensure `cron_sentinel_directory` exists and is writable:
   ```bash
   mkdir -p ~/.openclaw/workspace/cron-sentinels/
   ```
5. **Deploy Scripts**: Create the `baker-reindex.sh` script (§5.1) at `~/breadcrumb-trail/scripts/` and apply execution permissions:
   ```bash
   chmod +x ~/breadcrumb-trail/scripts/baker-reindex.sh
   ```
6. **Update Cron**: Align all system schedule entries to the Unified Schedule in Section 2. Ensure every cron job sources `sentinel-guard.sh` (§2.1) or implements equivalent sentinel logic.
7. **Cross-Register Agents**: Create `.crumb.md` files for the `memory-miner`, `crm-miner`, `soul-keeper`, etc., so they appear in the tool index.
8. **Trigger Reindex**: Run `baker-reindex.sh all` manually once to index the new agent breadcrumbs and verify REINDEX_RESULT output.
9. **Commission**: Verify end-to-end by running each cron job manually, confirming sentinel creation/cleanup, and checking that no budget violations are flagged.
