# OpenClaw Lab Manager — System Health & Operations Spec

*Version: 1.0 | Spec ID: `openclaw-lab-manager`*
*Audience: OpenClaw operators running one or more of the Persistent Memory, Breadcrumb System, or Integration Bridge specs*
*License: Share freely.*

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Design Principles](#2-design-principles)
3. [Prerequisites](#3-prerequisites)
4. [Agent Definition](#4-agent-definition)
5. [What the Lab Manager Measures](#5-what-the-lab-manager-measures)
6. [Health Report Format](#6-health-report-format)
7. [Measurement Methods](#7-measurement-methods)
8. [Tuning Recommendations Engine](#8-tuning-recommendations-engine)
9. [Verification Tests](#9-verification-tests)
10. [Schedule](#10-schedule)
11. [Build Order](#11-build-order)

---

## 1. Purpose

The Persistent Memory, Breadcrumb System, and Integration Bridge specs define *what to build*. This spec defines *how to know if it's working.*

After deployment, three questions go unanswered without active measurement:

1. **Is the system healthy?** Are cron jobs running, files being written, agents producing output on schedule?
2. **Is the system effective?** Are agents actually using breadcrumbs before tasks? Are recall failures decreasing over time? Is the Context Assembler loading relevant content?
3. **Is the system tuned?** Are the token budgets right? Are cron windows sufficient? Are staleness thresholds calibrated to real usage patterns?

The Lab Manager is a single agent that answers all three by reading the logs and artifacts the other systems already produce, running periodic verification tests, and generating a structured health report with actionable tuning recommendations.

### What This Spec Does Not Do

- Does not add new file formats, storage layers, or extension points
- Does not modify the other three specs
- Does not create a dashboard or UI (the report is a markdown file; build a dashboard later if you want one)
- Does not take autonomous corrective action (it recommends; the operator or agents act)

---

## 2. Design Principles

1. **Observe, don't interfere.** The Lab Manager is read-only against the systems it monitors. It writes only to its own report file and log. It never modifies CONFIG.md, breadcrumbs, memory files, or agent SOULs directly.
2. **Measure what matters.** Not every metric is actionable. The Lab Manager tracks metrics that lead to CONFIG.md changes, behavioral corrections, or architectural decisions.
3. **Recommend, don't mandate.** Every recommendation includes the evidence that supports it and the CONFIG.md parameter to change. The operator decides.
4. **No new complexity.** One agent, one cron job, one report file. Consumes existing logs and artifacts. Adds no dependencies to the other specs.
5. **Degrade to useful.** If only the Memory spec is deployed, the Lab Manager reports on memory health. If only Breadcrumbs are deployed, it reports on breadcrumb health. If all three plus the Bridge are deployed, it reports on everything. It adapts to what's present.

---

## 3. Prerequisites

### Required

| Component | Purpose | Notes |
|-----------|---------|-------|
| OpenClaw instance | Agent runtime | Any supported platform |
| At least one of: Persistent Memory spec, Breadcrumb System spec | Something to monitor | Lab Manager adapts to what's deployed |
| Cron scheduling | Weekly report generation | OpenClaw native |

### What the Lab Manager Reads (All Read-Only)

**From Persistent Memory spec:**

| Source | What It Reveals |
|--------|----------------|
| `memory/YYYY-MM-DD.md` | Daily event volume, system warnings, fallback events |
| `CORRECTIONS.md` | Recall failure frequency and recency |
| `CONFLICTS.md` | Conflict detection rate, resolution method distribution |
| `COMPACTION_LOG.md` | Memory hygiene effectiveness, deletion volume |
| `CHANGELOG.md` | Soul evolution frequency and type distribution |
| `MEMORY.md` | Total entry count, confidence distribution, tag distribution |
| `Archive/` | CRM profile count, staleness distribution, entity coverage |
| `CONFIG.md` | Current tuning parameters (for comparison against recommendations) |

**From Breadcrumb System spec:**

| Source | What It Reveals |
|--------|----------------|
| `~/breadcrumb-trail/baker.log` | Baker run success/failure, discovery rate, stale count, undocumented components |
| `~/breadcrumb-trail/index.json` | Total breadcrumbs, status distribution, token_estimate distribution |
| `~/breadcrumb-trail/recipe.json` | Capability count, mount status, flagged capabilities |
| `*.crumb.md` files | Individual breadcrumb staleness (updated date vs. source modification date) |

**From Integration Bridge:**

| Source | What It Reveals |
|--------|----------------|
| `CONFIG.md` extension points | Whether Bridge is active, current budget allocations |
| `baker.log` recall-triggered entries | Non-memory recall failure routing frequency |
| `ChatMessage` table (if accessible) | Heartbeat trigger hit rates by pattern group |

---

## 4. Agent Definition

**Name:** `lab-manager`
**Location:** `agents/lab-manager/`
**Schedule:** Cron — weekly (default: Sunday 10:00) + on-demand invocation

### SOUL.md

```markdown
# Lab Manager

## Role
System Health & Operations Analyst

## Mission
Monitor the health, effectiveness, and tuning of all deployed OpenClaw
infrastructure specs. Produce a weekly health report with evidence-based
recommendations. You are the quality assurance layer — the agent that
answers "is this actually working?"

You do not fix problems. You identify them, measure them, and recommend
specific changes with evidence. The operator and other agents act on your
recommendations.

## What You Monitor

Adapt to what is deployed. Check for the existence of each system's files
before attempting to measure it.

### System Detection
- If ~/breadcrumb-trail/recipe.json exists → Breadcrumb System is deployed
- If MEMORY.md exists in the workspace → Persistent Memory is deployed
- If CONFIG.md contains external_context_budget > 0 → Integration Bridge is active

### Health Metrics (Is it running?)
For each deployed system, verify:
- Cron jobs produced expected output since last report
- No error states persisting longer than 24 hours
- All expected files exist and are non-empty
- Mounts are accessible (if applicable)

### Effectiveness Metrics (Is it working?)
- Recall failure rate: count of CORRECTIONS.md entries per period (trending down = good)
- Breadcrumb coverage: ratio of documented vs. undocumented components (trending up = good)
- Standing Directive compliance: evidence of recipe.json reads before task execution
- Context Assembler utilization: are token budget slots being used or wasted?
- Conflict resolution rate: auto-resolved vs. flagged-for-operator ratio
- CRM staleness: percentage of profile fields in fresh vs. stale vs. expired status

### Tuning Metrics (Is it calibrated?)
- Token budget utilization per slot (is any slot consistently truncating? consistently underused?)
- Cron timing gaps (did any jobs overlap? did any jobs fail to start?)
- Breadcrumb token_estimate distribution (are most crumbs loadable within budget?)
- Memory confidence distribution (healthy = bell curve centered around 0.6-0.8; unhealthy = bimodal at 0.2 and 1.0)
- Correction archive rate (are corrections being archived on schedule?)

## How You Generate Reports

1. Detect which systems are deployed (see System Detection above)
2. For each deployed system, gather the Health, Effectiveness, and Tuning metrics
3. Compare current metrics against the previous report (if one exists) to identify trends
4. Generate recommendations for any metric that is outside expected range
5. Write the report to the designated output location
6. [Enhanced] Send a summary digest to the operator via configured messaging channel

## Rules
1. NEVER modify any file outside your own report and log. You are read-only against all other systems.
2. NEVER recommend disabling a system. You can recommend tuning parameters within a system.
3. Every recommendation must cite specific evidence (log entries, counts, timestamps).
4. If you cannot measure a metric because the data source is missing or unreadable, report it as "UNMEASURABLE — [reason]" rather than guessing.
5. Compare against the previous report when available. Trends matter more than absolutes.
6. Keep the report scannable. Operators will skim it. Lead with status, follow with evidence.
7. Flag anything that needs operator attention with a clear action item.

## Report Delivery

Primary: Write full report to ~/lab-reports/YYYY-MM-DD-health-report.md
Secondary: [Enhanced] Send summary digest via messaging channel (if configured)
Log: Append run metadata to ~/lab-reports/lab-manager.log
```

### Inputs / Outputs

| Direction | Source/Destination | Type | Notes |
|-----------|--------------------|------|-------|
| **Reads** | `memory/YYYY-MM-DD.md` | Files | Daily event logs (last 7 days) |
| **Reads** | `MEMORY.md` | File | Entry count, confidence distribution |
| **Reads** | `CORRECTIONS.md` | File | Recall failure count and recency |
| **Reads** | `CONFLICTS.md` | File | Conflict count and resolution types |
| **Reads** | `COMPACTION_LOG.md` | File | Deletion/archival volume |
| **Reads** | `CHANGELOG.md` | File | Soul evolution entries |
| **Reads** | `Archive/` | Files | CRM profile count and staleness |
| **Reads** | `CONFIG.md` | File | Current tuning parameters |
| **Reads** | `~/breadcrumb-trail/baker.log` | Log | Baker run results |
| **Reads** | `~/breadcrumb-trail/index.json` | JSON | Breadcrumb inventory |
| **Reads** | `~/breadcrumb-trail/recipe.json` | JSON | Capability inventory |
| **Reads** | `~/lab-reports/` | Files | Previous reports (for trend comparison) |
| **Writes** | `~/lab-reports/YYYY-MM-DD-health-report.md` | File | Weekly health report |
| **Writes** | `~/lab-reports/lab-manager.log` | Log | Run metadata |
| **Writes** | Messaging channel | Notification | [Enhanced] Summary digest |

---

## 5. What the Lab Manager Measures

### Metric Catalog

Each metric includes: what it measures, where the data comes from, what "healthy" looks like, and what triggers a recommendation.

#### Memory System Metrics

| Metric | Source | Healthy Range | Recommendation Trigger |
|--------|--------|--------------|----------------------|
| **Recall failures / week** | `CORRECTIONS.md` entry count | Trending down or stable at < 3/week | 5+ in a week, or trending up over 3 consecutive reports |
| **Recall failure classification** | `CORRECTIONS.md` classification field | Majority are `memory` type | > 30% classified as `external` suggests breadcrumb coverage gap |
| **Memory entry count** | `MEMORY.md` line/entry count | Growing slowly (5-20 new entries/week) | Flat (nothing being mined) or explosive (> 50/week, dedup may be failing) |
| **Confidence distribution** | `MEMORY.md` confidence values | Bell curve, median 0.6-0.8 | Bimodal (clustered at floor and ceiling) suggests scoring dysfunction |
| **CRM profile staleness** | `Archive/` `last_verified` fields | > 70% fresh | > 40% stale or expired |
| **Conflict resolution rate** | `CONFLICTS.md` resolution field | > 60% auto-resolved | > 50% flagged for operator (rules may need tuning) |
| **Compaction health** | `COMPACTION_LOG.md` recent entries | Entries present for each weekly run | Missing entries = compactor didn't run or had nothing to do |
| **Soul evolution frequency** | `CHANGELOG.md` entry count | 1-5 changes/week | 0 (soul keeper not running or not finding patterns) or > 10 (too volatile) |
| **Context Assembler fallbacks** | `memory/` files tagged `#system-warning` | 0 per week | Any fallback event warrants investigation |

#### Breadcrumb System Metrics

| Metric | Source | Healthy Range | Recommendation Trigger |
|--------|--------|--------------|----------------------|
| **Breadcrumb coverage** | `baker.log` NEEDS BREADCRUMB count | Trending to 0 | Stable or growing undocumented component count |
| **Stale breadcrumbs** | `baker.log` STALE count | < 10% of total | > 20% stale |
| **Baker run consistency** | `baker.log` timestamps | 4 runs/day with expected spacing | Missing runs, irregular timing, or error entries |
| **Deprecated breadcrumb age** | `index.json` deprecated entries | Deprecated items move to archive within 90 days | Deprecated items lingering past retention period |
| **Recipe capability health** | `recipe.json` status fields | All capabilities `active` or `experimental` | Any capability flagged as `broken` or `deprecated` |
| **Token estimate distribution** | `index.json` token_estimate values | Median < 500, 90th percentile < 800 | Median > 600 (crumbs are too verbose for budget-constrained loading) |
| **Mount availability** | `recipe.json` mounts + `baker.log` | All mounts online | Any mount offline for > 24 hours |

#### Integration Bridge Metrics (only when Bridge is active)

| Metric | Source | Healthy Range | Recommendation Trigger |
|--------|--------|--------------|----------------------|
| **External context utilization** | Context Assembler behavior | Budget consistently 50-90% used | < 20% used (budget too high, reclaim for other slots) or consistently truncating (budget too low) |
| **Recall routing volume** | `baker.log` recall-triggered entries | Low (< 3/week) | High volume suggests agents aren't checking recipe.json pre-flight |
| **Breadcrumb trigger hits** | `ChatMessage` priority flags | Decreasing over time | Stable or increasing = agents not building the breadcrumb habit |
| **CRM exclusion effectiveness** | `Archive/Things/` new profiles | No duplicate profiles for recipe.json capabilities | New Thing profiles that match recipe.json keys |
| **Cron overlap incidents** | `baker.log` + `memory/` timestamps | 0 | Any overlap between Baker and Memory agent write windows |

#### Standing Directive Compliance

| Metric | Source | Healthy Range | Recommendation Trigger |
|--------|--------|--------------|----------------------|
| **Pre-flight check rate** | Conversation logs / ChatMessage | Agents reading recipe.json before tasks | Evidence of tasks executed without recipe check (operator had to redirect) |
| **Search-before-ask rate** | CORRECTIONS.md + conversation context | Decreasing operator-answered questions over time | Stable or increasing "I had to tell the bot" patterns |
| **Breadcrumb creation on gap** | `baker.log` NEEDS BREADCRUMB trend | Undocumented count decreasing | Gaps persist across multiple reports without new crumbs being created |

---

## 6. Health Report Format

The report is a structured markdown file designed to be scanned in under 2 minutes.

```markdown
# Lab Manager Health Report
**Date:** YYYY-MM-DD
**Period:** YYYY-MM-DD to YYYY-MM-DD (7 days)
**Systems detected:** Persistent Memory ✅ | Breadcrumb System ✅ | Integration Bridge ✅

---

## Overall Status: 🟢 HEALTHY | 🟡 NEEDS ATTENTION | 🔴 ACTION REQUIRED

[One sentence summary of the most important finding]

---

## System Health (Is it running?)

| System | Component | Status | Last Activity | Notes |
|--------|-----------|--------|---------------|-------|
| Memory | Memory Miner | ✅ Running | YYYY-MM-DD 06:00 | 7/7 daily runs |
| Memory | CRM Miner | ✅ Running | YYYY-MM-DD 06:30 | 7/7 daily runs |
| Memory | Soul Keeper | ✅ Running | YYYY-MM-DD 02:30 | 7/7 daily runs |
| Memory | Recall Validator | ✅ Running | YYYY-MM-DD | 2 triggers this week |
| Memory | Conflict Detector | ✅ Running | YYYY-MM-DD 07:00 | 7/7 daily runs |
| Memory | Memory Compactor | ✅ Running | YYYY-MM-DD 03:00 | 1/1 weekly run |
| Breadcrumb | Baker | ✅ Running | YYYY-MM-DD 23:00 | 28/28 runs |
| Bridge | Cron coordination | ✅ No overlaps | — | Clean week |

---

## Effectiveness (Is it working?)

### Memory System
| Metric | This Week | Last Week | Trend | Status |
|--------|-----------|-----------|-------|--------|
| Recall failures | N | N | ↓↑→ | ✅🟡🔴 |
| Memory entries (total) | N | N | — | ✅🟡🔴 |
| CRM profiles fresh | N% | N% | ↓↑→ | ✅🟡🔴 |
| Conflicts auto-resolved | N% | N% | ↓↑→ | ✅🟡🔴 |
| Soul changes | N | N | — | ✅🟡🔴 |
| Context Assembler fallbacks | N | N | ↓↑→ | ✅🟡🔴 |

### Breadcrumb System
| Metric | This Week | Last Week | Trend | Status |
|--------|-----------|-----------|-------|--------|
| Undocumented components | N | N | ↓↑→ | ✅🟡🔴 |
| Stale breadcrumbs | N | N | ↓↑→ | ✅🟡🔴 |
| Broken capabilities | N | N | — | ✅🟡🔴 |
| Baker errors | N | N | — | ✅🟡🔴 |

### Agent Behavior
| Metric | This Week | Last Week | Trend | Status |
|--------|-----------|-----------|-------|--------|
| Pre-flight recipe checks | Evidence: Y/N | — | — | ✅🟡🔴 |
| Search-before-ask compliance | Evidence: Y/N | — | — | ✅🟡🔴 |
| Breadcrumbs created after gap | N | N | ↓↑→ | ✅🟡🔴 |

---

## Tuning (Is it calibrated?)

### Token Budget Utilization
| Slot | Budget | Avg Used | Truncation Events | Recommendation |
|------|--------|----------|-------------------|----------------|
| Identity | 800 | N | N | — |
| Operator Profile | 600 | N | N | — |
| Corrections | 400 | N | N | — |
| Recent Memory | 1200 | N | N | — |
| Semantic Recall | 1500 | N | N | — |
| CRM Profiles | N | N | N | — |
| Long-Term Memory | N | N | N | — |
| External Context | N | N | N | — |

### Breadcrumb Token Distribution
| Percentile | token_estimate |
|------------|---------------|
| Median | N tokens |
| 75th | N tokens |
| 90th | N tokens |
| Max | N tokens |

### Cron Timing
| Window | Expected Gap | Actual Gap | Status |
|--------|-------------|------------|--------|
| Baker → Memory Miner | 60 min | N min | ✅🟡🔴 |
| Memory Compactor → Baker (Sunday) | 300 min | N min | ✅🟡🔴 |

---

## Recommendations

### 🔴 Action Required
[Numbered list of critical recommendations with evidence and CONFIG.md parameter to change. Empty if none.]

### 🟡 Suggested Tuning
[Numbered list of non-critical optimizations with evidence. Empty if none.]

### ✅ Working Well
[Brief list of things that are working as designed — positive reinforcement.]

---

## Raw Data References

[File paths and line numbers for every data point cited in this report,
so the operator can verify independently.]

---

*Report generated by Lab Manager | Next report: YYYY-MM-DD*
```

### Summary Digest Format (Enhanced — messaging channel)

```
🔬 Lab Manager — YYYY-MM-DD
Status: 🟢 HEALTHY | 🟡 NEEDS ATTENTION | 🔴 ACTION REQUIRED
• [One-line summary of most important finding]
• Recall failures: N (↓↑→ from last week)
• Undocumented components: N
• Recommendations: N action required, N suggested
Full report: ~/lab-reports/YYYY-MM-DD-health-report.md
```

---

## 7. Measurement Methods

### How the Lab Manager Gathers Each Metric

The Lab Manager does not require any new instrumentation. Every metric is derived from files the other systems already produce. Here is how each measurement is taken.

#### Recall Failure Count

```bash
# Count entries in CORRECTIONS.md from the last 7 days
# CORRECTIONS.md entries start with a timestamp line: ## YYYY-MM-DDTHH:MM:SSZ
grep -c "^## $(date -d '7 days ago' +%Y)" CORRECTIONS.md
# More precise: parse timestamps and count those within the reporting window
```

#### Memory Entry Count and Confidence Distribution

```bash
# Count entries in MEMORY.md (each entry has a confidence value)
grep -c "confidence" MEMORY.md

# Extract confidence values for distribution analysis
grep -oP 'confidence:\s*\K[0-9.]+' MEMORY.md | sort -n
```

#### CRM Profile Staleness

```bash
# For each profile in Archive/People/, Archive/Places/, Archive/Things/:
# Parse last_verified timestamps, compare against staleness_threshold_days
find Archive/ -name "*.md" -exec grep -l "last_verified" {} \;
# Count fresh vs. stale vs. expired per the rules in the Memory spec Section 9
```

#### Baker Run Consistency

```bash
# Count BAKER RUN entries in baker.log from the last 7 days
grep -c "=== BAKER RUN:" ~/breadcrumb-trail/baker.log
# Verify expected count: 4/day × 7 days = 28 (weekday schedule)
```

#### Breadcrumb Coverage

```bash
# From baker.log, count NEEDS BREADCRUMB flags in most recent run
grep "NEEDS BREADCRUMB" ~/breadcrumb-trail/baker.log | tail -20
```

#### Token Estimate Distribution

```bash
# From index.json
cat ~/breadcrumb-trail/index.json | jq '[.breadcrumbs[].token_estimate] | sort'
cat ~/breadcrumb-trail/index.json | jq '[.breadcrumbs[].token_estimate] | (length) as $n | sort | .[$n/2]'
```

#### Cron Overlap Detection

```bash
# Compare timestamps in baker.log against memory/ file write times
# If any Baker Phase 2 (Validate) timestamp overlaps with a Memory Miner
# write timestamp (within 5 minutes), flag as overlap
```

#### Standing Directive Compliance

This is the hardest metric to measure automatically. Methods:

1. **Indirect evidence from CORRECTIONS.md:** If a recall failure is classified as `external` (tool/capability), it means the agent didn't check recipe.json. Track frequency.
2. **Indirect evidence from baker.log:** If recall-triggered reindex entries appear, it means agents are forgetting capabilities. Track frequency.
3. **Direct evidence from ChatMessage:** If accessible, search for messages where the operator redirected the agent to a breadcrumb ("we already have a script for that"). Track frequency.
4. **Proxy metric:** Undocumented component count trend. If agents are following Rule 2 (search-before-ask) and Rule 4 (breadcrumb what you build), the undocumented count should trend to zero.

---

## 8. Tuning Recommendations Engine

The Lab Manager generates recommendations by comparing measured values against expected ranges. Each recommendation follows this format:

```markdown
### Recommendation: [Title]
**Severity:** 🔴 Action Required | 🟡 Suggested Tuning
**Evidence:** [Specific metric, value, source file, line numbers]
**Trend:** [Comparison to previous report if available]
**Recommendation:** Change `parameter_name` in CONFIG.md from `current_value` to `recommended_value`
**Rationale:** [Why this change should help]
```

### Common Recommendations

| Trigger Condition | Recommendation |
|-------------------|---------------|
| External context slot consistently truncating | Increase `external_context_budget` by 200 tokens. Consider increasing `context_budget_tokens` total to avoid stealing from other slots. |
| External context slot < 20% utilized | Decrease `external_context_budget` to 400 (recipe.json summary only). Reallocate freed tokens to `crm_tokens` or `longterm_memory_tokens`. |
| CRM profiles > 40% stale | Decrease `staleness_threshold_days` from 90 to 60 to force more frequent verification prompts. |
| Recall failures trending up | Check if CORRECTIONS.md entries share a common topic. If yes, the Memory Miner may be failing to store that category. Inspect Memory Miner's deduplication — it may be merging away corrections. |
| Recall failures > 30% classified as `external` | Breadcrumb coverage gap. Cross-reference with baker.log NEEDS BREADCRUMB list. Create missing breadcrumbs. |
| Memory entry count flat (0 new entries/week) | Memory Miner may not be running or may be over-filtering. Check cron schedule. Check heartbeat trigger patterns — they may be too narrow. |
| Memory entry count explosive (> 50/week) | Memory Miner deduplication may be failing. Check `vector_dedup_threshold` (if Enhanced) or keyword matching logic (if Core). |
| Confidence distribution bimodal | Confidence scoring dynamics may need recalibration. Check `confidence_decay_amount` and `confidence_boost_per_mention` — decay may be too aggressive or boost too generous. |
| Soul changes > 10/week | Soul Keeper may be making changes on insufficient evidence. Review CHANGELOG.md for entries with weak evidence citations. Consider adding "require 3+ occurrences before encoding as principle" to Soul Keeper's rules. |
| Soul changes = 0 for 3+ weeks | Soul Keeper may not be running, or recent conversations may lack behavioral signal. Check cron. If running, this may be fine — not every week produces growth. |
| Baker errors > 0 | Investigate immediately. Baker errors usually mean filesystem issues, permission problems, or corrupted crumb files. |
| Breadcrumb median token_estimate > 600 | Breadcrumbs are getting verbose. Agents may be writing overly detailed crumbs that blow context budgets. Consider adding a size guideline to the Standing Directive. |
| Cron overlap detected | Adjust timing in the unified schedule. Increase gap between Baker completion and Memory Miner start. |
| Undocumented components stable across 3+ reports | Standing Directive Rule 2 (search-before-ask) is not being followed, or Rule 4 (breadcrumb what you build) is being skipped. Reinforce in agent prompts. |
| Context Assembler fallback events > 0 | Investigate the failed resource. If it's a mount, check connectivity. If it's a file, check permissions. If it's Vector DB, check service health. |

---

## 9. Verification Tests

The Lab Manager runs the verification tests defined in all three specs' Build Orders. These tests are binary pass/fail and appear in the Health section of the report.

### Memory System Tests

| Test | Method | Pass Condition |
|------|--------|---------------|
| Context Assembler loads | Start a session, inspect injected context | SOUL.md, USER.md, and at least one memory slot loaded |
| Heartbeat fires on recall trigger | Send "I already told you X" | CORRECTIONS.md receives new entry within 60 seconds |
| Memory Miner produces output | Check MEMORY.md modification date | Modified within last 24 hours (on a day with conversations) |
| CRM Miner produces output | Check Archive/ modification dates | At least one profile modified in last 7 days (if entities were mentioned) |
| Soul Keeper produces output | Check CHANGELOG.md modification date | Modified within last 7 days |

### Breadcrumb System Tests

| Test | Method | Pass Condition |
|------|--------|---------------|
| Baker runs on schedule | Check baker.log timestamps | Expected number of runs in reporting period |
| recipe.json is valid | Parse JSON, check for required fields | Valid JSON, at least one capability entry |
| index.json matches filesystem | Compare index entries against actual .crumb.md files | All indexed files exist; no orphaned crumbs missing from index |
| Cold start test | Fresh session → task requiring documented capability | Agent finds capability via recipe.json without operator help |

### Integration Bridge Tests (only when Bridge is active)

| Test | Method | Pass Condition |
|------|--------|---------------|
| External context loads at session start | Inspect Context Assembler output | recipe.json summary present in context |
| Recall routing works | Trigger tool-related recall failure | baker-reindex.sh invoked, not MEMORY.md |
| CRM exclusion works | Check Archive/ for recipe.json entities | No full profiles for entities in recipe.json |
| Cron schedule clean | Compare all log timestamps | No overlapping write windows |

---

## 10. Schedule

| Run Type | Frequency | Default Time | Notes |
|----------|-----------|-------------|-------|
| **Weekly report** | Weekly | Sunday 10:00 | After all maintenance jobs complete. Produces full health report. |
| **On-demand** | Manual | — | Operator invokes: "run lab manager" or "health check" |

### Why Sunday 10:00

The unified cron schedule (from the Integration Bridge) has all weekly maintenance completing by 08:00 Sunday. Baker's Sunday run is at 08:00. The Lab Manager runs at 10:00, after everything else has finished, so it measures a complete week of data including the Sunday maintenance results.

### Cron Configuration

```bash
# Lab Manager — weekly report
0 10 * * 0 /path/to/lab-manager-run.sh
```

---

## 11. Build Order

### Step 1: Create Directory Structure

```bash
mkdir -p ~/lab-reports
```

### Step 2: Deploy the Lab Manager Agent

Install the SOUL.md from Section 4 into `agents/lab-manager/SOUL.md`.

### Step 3: Create the Lab Manager Breadcrumb

If the Breadcrumb System is deployed, create `agents/lab-manager/agent.lab-manager.crumb.md`:

```markdown
---
id: agent_lab-manager
name: Lab Manager
type: agent
status: active
created: YYYY-MM-DD
updated: YYYY-MM-DD
owner: shared
source: ~/agents/lab-manager/
token_estimate: 0
tags: [operations, health, monitoring, tuning]
deps: [openclaw, jq]
cron: "Sunday 10:00"
---

# Purpose
Monitor health, effectiveness, and tuning of all deployed OpenClaw infrastructure
specs and produce weekly reports with actionable recommendations.

# How It Works
1. Detects which systems are deployed (Memory, Breadcrumbs, Bridge)
2. Reads logs and artifacts from each detected system (read-only)
3. Measures health, effectiveness, and tuning metrics
4. Compares against previous report for trends
5. Generates structured health report with recommendations
6. Writes report to ~/lab-reports/

# Usage
## Commands
\```bash
# Automatic (weekly cron)
# Runs Sunday 10:00

# Manual trigger
openclaw run lab-manager
\```

## Outputs
| Output | Location | Format |
|--------|----------|--------|
| Health report | ~/lab-reports/YYYY-MM-DD-health-report.md | Markdown |
| Run log | ~/lab-reports/lab-manager.log | Log |

# Gotchas
- 📌 Read-only agent — never modifies files from other systems
- 📌 Adapts to what's deployed — skip metrics for systems that aren't present
- 📌 Runs after all other Sunday maintenance (10:00, after Baker at 08:00)
- ⚠️ Standing Directive compliance is measured indirectly — accuracy depends
  on quality of CORRECTIONS.md classification and baker.log entries

# Changelog
- **YYYY-MM-DD**: Created
```

### Step 4: Schedule the Cron Job

Add the Lab Manager to the system crontab: Sunday 10:00.

### Step 5: Run Baker

If the Breadcrumb System is deployed, run Baker manually to index the new Lab Manager breadcrumb.

### Step 6: Run Lab Manager Manually

Execute the Lab Manager once to generate the first baseline report. This report will have no trend data (no previous report to compare against) but will establish the baseline for all future comparisons.

### Step 7: Verify

- Check `~/lab-reports/` for the generated report file
- Open the report and confirm it detected the correct deployed systems
- Confirm metrics are populated (not all UNMEASURABLE)
- If using the messaging channel digest, confirm delivery

---

## Summary

| Aspect | Value |
|--------|-------|
| **Agent count** | 1 |
| **Cron jobs** | 1 (Sunday 10:00) |
| **New file formats** | 0 (uses existing logs and artifacts) |
| **New storage layers** | 0 (one output directory for reports) |
| **New extension points** | 0 (consumes, doesn't extend) |
| **Impact on other specs** | None (read-only) |
| **Adapts to deployment** | Yes — measures only what's deployed |
| **Report time to scan** | < 2 minutes |

---

*This spec adds an operations layer to the OpenClaw infrastructure without increasing the complexity of the systems it monitors. One agent, one report, one cron job. If you remove it, nothing else breaks.*
