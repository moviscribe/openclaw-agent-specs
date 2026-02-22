# OpenClaw Lab Manager — System Health & Operations Spec

*Version: 1.1 | Spec ID: `openclaw-lab-manager`*
*Audience: OpenClaw operators running one or more of the Persistent Memory, Breadcrumb System, or Integration Bridge specs*
*License: Share freely.*

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| **1.1** | 2026-02-22 | Added Daily Heartbeat Script (§10.1). Expanded Standing Directive metrics with indirect/experimental indicators, cron-sentinel monitoring, stale `.lock` file detection, and correction pipeline health tracking (§4, §5). Added `agent.lab-manager.crumb.md` breadcrumb requirement (§11 Step 3). Added daily heartbeat cron entry alongside weekly full report (§10). |
| **1.0** | — | Initial release. |

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
    1. [Daily Heartbeat Script](#101-daily-heartbeat-script)
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
4. **No new complexity.** One agent, two cron jobs (daily heartbeat + weekly report), one report file. Consumes existing logs and artifacts. Adds no dependencies to the other specs.
5. **Degrade to useful.** If only the Memory spec is deployed, the Lab Manager reports on memory health. If only Breadcrumbs are deployed, it reports on breadcrumb health. If all three plus the Bridge are deployed, it reports on everything. It adapts to what's present.
6. **Two-tier monitoring.** *(v1.1)* The daily heartbeat catches critical failures fast (pass/fail, < 60 seconds). The weekly report provides depth and trend analysis. Neither replaces the other.

---

## 3. Prerequisites

### Required

| Component | Purpose | Notes |
|-----------|---------|-------|
| OpenClaw instance | Agent runtime | Any supported platform |
| At least one of: Persistent Memory spec, Breadcrumb System spec | Something to monitor | Lab Manager adapts to what's deployed |
| Cron scheduling | Daily heartbeat + weekly report generation | OpenClaw native |

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

**From System Infrastructure:** *(v1.1)*

| Source | What It Reveals |
|--------|----------------|
| Cron sentinel files (e.g., `/tmp/*.sentinel`) | Whether cron jobs actually executed recently |
| `/tmp/*.lock` files | Stale locks from crashed or hung agents |
| `CORRECTIONS.md` timestamps + resolution status | Correction pipeline throughput and cool-down health |

---

## 4. Agent Definition

**Name:** `lab-manager`
**Location:** `agents/lab-manager/`
**Schedule:** Cron — daily heartbeat (default: 07:00) + weekly full report (default: Sunday 10:00) + on-demand invocation
**Breadcrumb:** `agents/lab-manager/agent.lab-manager.crumb.md` (required; see §11 Step 3)

### SOUL.md

```markdown
# Lab Manager

## Role
System Health & Operations Analyst

## Mission
Monitor the health, effectiveness, and tuning of all deployed OpenClaw
infrastructure specs. Produce a weekly health report with evidence-based
recommendations and a daily heartbeat with pass/fail status checks.
You are the quality assurance layer — the agent that answers
"is this actually working?"

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
- Cron sentinel files are present and recent (v1.1)
- No stale .lock files lingering from crashed agents (v1.1)

### Effectiveness Metrics (Is it working?)
- Recall failure rate: count of CORRECTIONS.md entries per period (trending down = good)
- Breadcrumb coverage: ratio of documented vs. undocumented components (trending up = good)
- Standing Directive compliance: evidence of recipe.json reads before task execution
  - **Indirect/experimental indicators (v1.1):** Where direct measurement is unavailable,
    the Lab Manager may observe indirect signals — operator correction frequency in
    conversation logs, recall-triggered reindex events in baker.log, and correction
    classification patterns in CORRECTIONS.md — to infer compliance. These are
    labeled "indirect" or "experimental" in reports and carry lower confidence.
- Context Assembler utilization: are token budget slots being used or wasted?
- Conflict resolution rate: auto-resolved vs. flagged-for-operator ratio
- CRM staleness: percentage of profile fields in fresh vs. stale vs. expired status
- Correction pipeline health (v1.1): corrections moving from "open" to "verified"
  within expected cool-down window

### Tuning Metrics (Is it calibrated?)
- Token budget utilization per slot (is any slot consistently truncating? consistently underused?)
- Cron timing gaps (did any jobs overlap? did any jobs fail to start?)
- Cron sentinel freshness (v1.1): are sentinel files being touched on schedule?
- Breadcrumb token_estimate distribution (are most crumbs loadable within budget?)
- Memory confidence distribution (healthy = bell curve centered around 0.6-0.8; unhealthy = bimodal at 0.2 and 1.0)
- Correction archive rate (are corrections being archived on schedule?)
- Stale .lock file count (v1.1): any lock files older than their expected timeout?

## How You Generate Reports

1. Detect which systems are deployed (see System Detection above)
2. For each deployed system, gather the Health, Effectiveness, and Tuning metrics
3. Compare current metrics against the previous report (if one exists) to identify trends
4. Generate recommendations for any metric that is outside expected range
5. Write the report to the designated output location
6. [Enhanced] Send a summary digest to the operator via configured messaging channel

## Daily Heartbeat (v1.1)

In addition to the weekly full report, run a lightweight daily heartbeat check
(agents/lab-manager/lab-manager-heartbeat.sh) that spot-checks critical metrics
and outputs pass/fail per check. See §10.1 for details.

## Rules
1. NEVER modify any file outside your own report and log. You are read-only against all other systems.
2. NEVER recommend disabling a system. You can recommend tuning parameters within a system.
3. Every recommendation must cite specific evidence (log entries, counts, timestamps).
4. If you cannot measure a metric because the data source is missing or unreadable, report it as "UNMEASURABLE — [reason]" rather than guessing.
5. Compare against the previous report when available. Trends matter more than absolutes.
6. Keep the report scannable. Operators will skim it. Lead with status, follow with evidence.
7. Flag anything that needs operator attention with a clear action item.
8. For indirect/experimental metrics (v1.1), clearly label confidence level and measurement method.

## Report Delivery

Primary: Write full report to ~/lab-reports/YYYY-MM-DD-health-report.md
Heartbeat: Write daily heartbeat to ~/lab-reports/heartbeat/YYYY-MM-DD-heartbeat.log
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
| **Reads** | `/tmp/*.sentinel` | Files | *(v1.1)* Cron sentinel timestamps |
| **Reads** | `/tmp/*.lock` | Files | *(v1.1)* Stale lock detection |
| **Writes** | `~/lab-reports/YYYY-MM-DD-health-report.md` | File | Weekly health report |
| **Writes** | `~/lab-reports/heartbeat/YYYY-MM-DD-heartbeat.log` | File | *(v1.1)* Daily heartbeat results |
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
| **Indirect compliance indicators** *(v1.1, experimental)* | CORRECTIONS.md classification patterns, baker.log reindex frequency, operator redirect frequency | Declining indirect failure signals | Stable or rising indirect signals across 3+ reports — suggests Standing Directive erosion |

#### Infrastructure Health Metrics *(v1.1)*

| Metric | Source | Healthy Range | Recommendation Trigger |
|--------|--------|--------------|----------------------|
| **Cron sentinel freshness** | `/tmp/*.sentinel` files (mtime) | Each sentinel touched within its expected schedule + 15 min grace | Any sentinel older than expected schedule × 2 |
| **Stale .lock files** | `/tmp/*.lock` files (mtime) | 0 stale locks | Any `.lock` file older than 2× its expected job duration (default: 2 hours). Indicates a crashed or hung agent left a lock behind. |
| **Correction pipeline throughput** *(v1.1)* | `CORRECTIONS.md` entry timestamps + resolution status | Corrections move to "verified" or "archived" within 7 days | > 3 corrections unresolved for 14+ days — pipeline may be stalled |
| **Correction cool-down health** *(v1.1)* | `CORRECTIONS.md` consecutive correction timestamps | ≥ 24 hours between corrections for the same topic | Multiple corrections for the same topic within 24 hours — indicates thrashing (correction applied but not sticking) |

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
| Infra | Cron sentinels | ✅ All fresh | YYYY-MM-DD 07:05 | N/N sentinels current |
| Infra | Lock files | ✅ No stale locks | — | 0 stale / N total |
| Infra | Daily heartbeat | ✅ Running | YYYY-MM-DD 07:00 | 7/7 daily runs |

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
| Correction pipeline (unresolved >14d) | N | N | ↓↑→ | ✅🟡🔴 |
| Correction cool-down violations | N | N | ↓↑→ | ✅🟡🔴 |

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
| Standing Directive (indirect) | Evidence: Y/N | — | ↓↑→ | ✅🟡🔴 |

### Infrastructure (v1.1)
| Metric | This Week | Last Week | Trend | Status |
|--------|-----------|-----------|-------|--------|
| Stale sentinels detected | N | N | ↓↑→ | ✅🟡🔴 |
| Stale lock files detected | N | N | ↓↑→ | ✅🟡🔴 |
| Heartbeat failures (daily) | N/7 | N/7 | ↓↑→ | ✅🟡🔴 |

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
| Heartbeat → Baker (daily) | 60 min | N min | ✅🟡🔴 |

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
• Stale locks: N | Stale sentinels: N
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

> **v1.1 Note — Indirect/Experimental Indicators:**
> Where direct measurement is impractical, the Lab Manager may use indirect indicators
> to infer Standing Directive compliance. These include:
>
> - **Correction classification patterns:** A rising share of `external`-classified corrections
>   suggests agents are not consulting breadcrumbs pre-flight.
> - **Reindex frequency in baker.log:** Frequent recall-triggered reindexes indicate agents are
>   forgetting capabilities they should already know.
> - **Operator redirect frequency:** If the ChatMessage table or conversation logs show increasing
>   "we already have X" redirects from the operator, compliance is eroding.
>
> These are labeled `(indirect)` or `(experimental)` in the health report. They carry lower
> confidence than direct measurements and should be corroborated before acting on them.

#### Cron Sentinel Monitoring *(v1.1)*

```bash
# Each cron job should touch a sentinel file on successful completion:
#   touch /tmp/<jobname>.sentinel
#
# Lab Manager checks sentinel freshness:
for sentinel in /tmp/*.sentinel; do
  job=$(basename "$sentinel" .sentinel)
  age_seconds=$(( $(date +%s) - $(stat -f %m "$sentinel" 2>/dev/null || stat -c %Y "$sentinel") ))
  expected_interval=86400  # default: 24 hours; override per job
  grace=900  # 15 minute grace period
  if [ "$age_seconds" -gt $(( expected_interval + grace )) ]; then
    echo "STALE SENTINEL: $job (age: ${age_seconds}s, expected: ${expected_interval}s)"
  else
    echo "OK: $job (age: ${age_seconds}s)"
  fi
done
```

**Convention:** Each cron job that wants Lab Manager monitoring should `touch /tmp/<jobname>.sentinel` as its final step on success. The Lab Manager does not *create* these files — it only reads them.

#### Stale Lock File Detection *(v1.1)*

```bash
# Detect .lock files older than their expected timeout (default: 2 hours)
LOCK_TIMEOUT=7200  # seconds
for lockfile in /tmp/*.lock; do
  [ -f "$lockfile" ] || continue
  age=$(( $(date +%s) - $(stat -f %m "$lockfile" 2>/dev/null || stat -c %Y "$lockfile") ))
  if [ "$age" -gt "$LOCK_TIMEOUT" ]; then
    pid=$(cat "$lockfile" 2>/dev/null)
    if [ -n "$pid" ] && ! kill -0 "$pid" 2>/dev/null; then
      echo "STALE LOCK: $lockfile (age: ${age}s, PID $pid not running — crashed agent)"
    else
      echo "LONG LOCK: $lockfile (age: ${age}s, PID $pid still running — may be hung)"
    fi
  fi
done
```

**What this catches:** When an agent crashes mid-task, it may leave a `.lock` file behind (per the Lock File protocol in AGENTS.md). The next run of that job skips execution, thinking it's already running. Stale lock detection identifies these orphaned locks so they can be cleaned up.

#### Correction Pipeline Health *(v1.1)*

```bash
# Count corrections that have been unresolved for >14 days
# Corrections in CORRECTIONS.md have a timestamp and optional status field
# Look for entries without "verified" or "archived" status older than 14 days

CUTOFF=$(date -d '14 days ago' +%Y-%m-%d 2>/dev/null || date -v-14d +%Y-%m-%d)
# Count entries before cutoff that lack resolution markers
grep -B1 "^## " CORRECTIONS.md | grep -v "verified\|archived" | \
  awk -v cutoff="$CUTOFF" '$0 ~ /^## [0-9]{4}-/ && $2 < cutoff {count++} END {print count+0}'
```

#### Correction Cool-Down Detection *(v1.1)*

```bash
# Detect corrections for the same topic occurring within 24 hours of each other
# This indicates "thrashing" — a correction was applied but didn't stick
# Parse CORRECTIONS.md for entries sharing a topic/key and check time deltas
```

**What "thrashing" means:** If the system corrects the same recall failure multiple times in rapid succession, the correction mechanism isn't working. Either the Memory Miner is overwriting the correction, deduplication is merging it away, or the original source of error hasn't been addressed.

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
| Stale sentinel detected *(v1.1)* | Cron job `[name]` has not run since `[timestamp]`. Check crontab for the job entry. Check system logs for cron daemon health. If the sentinel file is missing entirely, the job may never have been instrumented — add `touch /tmp/<jobname>.sentinel` as its final step. |
| Stale .lock file detected *(v1.1)* | Lock file `[path]` is `[age]` old and PID `[pid]` is not running. The owning agent likely crashed. Remove the lock file to unblock future runs: `rm [path]`. Investigate the crash by checking the agent's log. |
| Correction thrashing detected *(v1.1)* | Topic `[topic]` has been corrected `[N]` times in `[period]`. The correction is not sticking. Check if Memory Miner deduplication is merging away the correction, or if the original source of error (e.g., a stale memory entry) hasn't been removed. |
| Correction pipeline stalled *(v1.1)* | `[N]` corrections have been unresolved for 14+ days. Review CORRECTIONS.md for entries that should have been verified and archived. The Recall Validator or operator may need to process the backlog. |

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

### Infrastructure Tests *(v1.1)*

| Test | Method | Pass Condition |
|------|--------|---------------|
| Cron sentinels present | Check for expected sentinel files in `/tmp/` | All configured sentinel files exist |
| Cron sentinels fresh | Compare sentinel mtime against expected schedule | All sentinels within expected interval + grace |
| No stale locks | Scan `/tmp/*.lock` for age > timeout | 0 stale lock files |
| Daily heartbeat runs | Check `~/lab-reports/heartbeat/` for today's file | Today's heartbeat log exists and shows pass/fail results |
| Lab Manager breadcrumb exists | Check `agents/lab-manager/agent.lab-manager.crumb.md` | File exists, valid frontmatter, status = active |

---

## 10. Schedule

| Run Type | Frequency | Default Time | Notes |
|----------|-----------|-------------|-------|
| **Daily heartbeat** *(v1.1)* | Daily | 07:00 | Lightweight pass/fail spot-check. Catches critical failures fast. |
| **Weekly report** | Weekly | Sunday 10:00 | After all maintenance jobs complete. Produces full health report with trends. |
| **On-demand** | Manual | — | Operator invokes: "run lab manager" or "health check" |

### Why Sunday 10:00

The unified cron schedule (from the Integration Bridge) has all weekly maintenance completing by 08:00 Sunday. Baker's Sunday run is at 08:00. The Lab Manager runs at 10:00, after everything else has finished, so it measures a complete week of data including the Sunday maintenance results.

### Why 07:00 Daily *(v1.1)*

The daily heartbeat runs early enough to catch overnight failures before the operator's workday begins. It completes in under 60 seconds and does not interfere with Baker or Memory Miner schedules (which typically run at off-hours or every 6 hours).

### Cron Configuration

```bash
# Lab Manager — daily heartbeat (v1.1)
0 7 * * * /path/to/agents/lab-manager/lab-manager-heartbeat.sh

# Lab Manager — weekly full report
0 10 * * 0 /path/to/lab-manager-run.sh
```

---

### 10.1. Daily Heartbeat Script

*(Added in v1.1)*

The daily heartbeat is a lightweight script that spot-checks critical system health and outputs pass/fail for each check. It runs in under 60 seconds and is designed to catch failures *between* weekly reports.

**Script location:** `agents/lab-manager/lab-manager-heartbeat.sh`
**Must be:** Executable (`chmod +x`)
**Output:** `~/lab-reports/heartbeat/YYYY-MM-DD-heartbeat.log`

#### What It Checks

The heartbeat checks a focused subset of critical metrics — things that, if broken, mean the system is not functioning:

| Check | What It Verifies | Pass Condition |
|-------|-----------------|----------------|
| **Memory Miner ran** | `memory/` has a file from today or yesterday | Recent daily file exists and is non-empty |
| **Baker ran** | `baker.log` has an entry from the last 24h | Recent BAKER RUN timestamp found |
| **No stale locks** | `/tmp/*.lock` files age check | 0 lock files older than 2 hours |
| **Cron sentinels fresh** | `/tmp/*.sentinel` mtime check | All sentinels within expected interval |
| **CORRECTIONS.md accessible** | File exists and is readable | File exists, size > 0 |
| **Breadcrumb index valid** | `index.json` parses as valid JSON | `jq . index.json` exits 0 |
| **Report directory writable** | Can write to `~/lab-reports/heartbeat/` | Test write succeeds |
| **No correction thrashing** | No same-topic corrections within 24h | 0 thrashing events detected |

#### Reference Implementation

```bash
#!/usr/bin/env bash
# Lab Manager — Daily Heartbeat (v1.1)
# Lightweight pass/fail health check. Runs daily via cron.
# Output: ~/lab-reports/heartbeat/YYYY-MM-DD-heartbeat.log

set -euo pipefail

DATE=$(date +%Y-%m-%d)
YESTERDAY=$(date -v-1d +%Y-%m-%d 2>/dev/null || date -d 'yesterday' +%Y-%m-%d)
HEARTBEAT_DIR="${HOME}/lab-reports/heartbeat"
LOGFILE="${HEARTBEAT_DIR}/${DATE}-heartbeat.log"
WORKSPACE="${HOME}/.openclaw/workspace"  # adjust to your workspace root

mkdir -p "$HEARTBEAT_DIR"

PASS=0
FAIL=0
WARN=0

check() {
  local name="$1"
  local result="$2"  # PASS, FAIL, or WARN
  local detail="$3"
  
  case "$result" in
    PASS) PASS=$((PASS + 1)); icon="✅" ;;
    FAIL) FAIL=$((FAIL + 1)); icon="❌" ;;
    WARN) WARN=$((WARN + 1)); icon="⚠️" ;;
  esac
  
  echo "${icon} ${result}: ${name} — ${detail}" >> "$LOGFILE"
}

# --- Header ---
{
  echo "# Lab Manager Daily Heartbeat"
  echo "**Date:** ${DATE}"
  echo "**Time:** $(date +%H:%M:%S)"
  echo "---"
  echo ""
} > "$LOGFILE"

# --- Check 1: Memory Miner ran ---
if [ -f "${WORKSPACE}/memory/${DATE}.md" ] || [ -f "${WORKSPACE}/memory/${YESTERDAY}.md" ]; then
  check "Memory Miner" "PASS" "Recent daily memory file found"
else
  check "Memory Miner" "FAIL" "No memory file for today or yesterday"
fi

# --- Check 2: Baker ran in last 24h ---
BAKER_LOG="${HOME}/breadcrumb-trail/baker.log"
if [ -f "$BAKER_LOG" ]; then
  if grep -q "=== BAKER RUN:.*${DATE}\|=== BAKER RUN:.*${YESTERDAY}" "$BAKER_LOG" 2>/dev/null; then
    check "Baker" "PASS" "Baker ran within last 24h"
  else
    check "Baker" "FAIL" "No Baker run found in last 24h"
  fi
else
  check "Baker" "WARN" "baker.log not found — Breadcrumb System may not be deployed"
fi

# --- Check 3: No stale lock files ---
STALE_LOCKS=0
LOCK_TIMEOUT=7200
for lockfile in /tmp/*.lock; do
  [ -f "$lockfile" ] || continue
  age=$(( $(date +%s) - $(stat -f %m "$lockfile" 2>/dev/null || stat -c %Y "$lockfile" 2>/dev/null || echo 0) ))
  if [ "$age" -gt "$LOCK_TIMEOUT" ]; then
    pid=$(cat "$lockfile" 2>/dev/null || echo "unknown")
    if [ "$pid" != "unknown" ] && ! kill -0 "$pid" 2>/dev/null; then
      STALE_LOCKS=$((STALE_LOCKS + 1))
    fi
  fi
done
if [ "$STALE_LOCKS" -eq 0 ]; then
  check "Lock files" "PASS" "No stale lock files detected"
else
  check "Lock files" "FAIL" "${STALE_LOCKS} stale lock file(s) — crashed agent likely"
fi

# --- Check 4: Cron sentinels fresh ---
STALE_SENTINELS=0
SENTINEL_TIMEOUT=90000  # 25 hours (daily jobs + grace)
for sentinel in /tmp/*.sentinel; do
  [ -f "$sentinel" ] || continue
  age=$(( $(date +%s) - $(stat -f %m "$sentinel" 2>/dev/null || stat -c %Y "$sentinel" 2>/dev/null || echo 0) ))
  if [ "$age" -gt "$SENTINEL_TIMEOUT" ]; then
    STALE_SENTINELS=$((STALE_SENTINELS + 1))
  fi
done
if [ "$STALE_SENTINELS" -eq 0 ]; then
  check "Cron sentinels" "PASS" "All sentinels fresh (or none configured)"
else
  check "Cron sentinels" "FAIL" "${STALE_SENTINELS} stale sentinel(s) — cron job(s) may have stopped"
fi

# --- Check 5: CORRECTIONS.md accessible ---
CORRECTIONS="${WORKSPACE}/CORRECTIONS.md"
if [ -f "$CORRECTIONS" ] && [ -s "$CORRECTIONS" ]; then
  check "CORRECTIONS.md" "PASS" "File exists and is non-empty"
elif [ -f "$CORRECTIONS" ]; then
  check "CORRECTIONS.md" "WARN" "File exists but is empty (may be normal if no corrections yet)"
else
  check "CORRECTIONS.md" "WARN" "File not found — Memory System may not be deployed"
fi

# --- Check 6: Breadcrumb index valid ---
INDEX="${HOME}/breadcrumb-trail/index.json"
if [ -f "$INDEX" ]; then
  if jq . "$INDEX" > /dev/null 2>&1; then
    check "Breadcrumb index" "PASS" "index.json is valid JSON"
  else
    check "Breadcrumb index" "FAIL" "index.json is invalid JSON — Baker may be corrupted"
  fi
else
  check "Breadcrumb index" "WARN" "index.json not found — Breadcrumb System may not be deployed"
fi

# --- Check 7: Report directory writable ---
if touch "${HEARTBEAT_DIR}/.write-test" 2>/dev/null && rm -f "${HEARTBEAT_DIR}/.write-test"; then
  check "Report directory" "PASS" "~/lab-reports/heartbeat/ is writable"
else
  check "Report directory" "FAIL" "Cannot write to ~/lab-reports/heartbeat/"
fi

# --- Check 8: No correction thrashing ---
if [ -f "$CORRECTIONS" ] && [ -s "$CORRECTIONS" ]; then
  # Simple heuristic: count corrections from today — >2 for same topic = thrashing
  TODAY_CORRECTIONS=$(grep -c "^## ${DATE}" "$CORRECTIONS" 2>/dev/null || echo 0)
  if [ "$TODAY_CORRECTIONS" -le 2 ]; then
    check "Correction cool-down" "PASS" "${TODAY_CORRECTIONS} correction(s) today (within normal range)"
  else
    check "Correction cool-down" "WARN" "${TODAY_CORRECTIONS} corrections today — possible thrashing"
  fi
else
  check "Correction cool-down" "PASS" "No corrections file to check (normal if Memory System not deployed)"
fi

# --- Summary ---
TOTAL=$((PASS + FAIL + WARN))
{
  echo ""
  echo "---"
  echo "## Summary"
  echo "- **Total checks:** ${TOTAL}"
  echo "- **Passed:** ${PASS} ✅"
  echo "- **Failed:** ${FAIL} ❌"
  echo "- **Warnings:** ${WARN} ⚠️"
  echo ""
  if [ "$FAIL" -gt 0 ]; then
    echo "**Overall: ❌ FAILURES DETECTED** — Review failed checks above."
  elif [ "$WARN" -gt 0 ]; then
    echo "**Overall: ⚠️ WARNINGS** — Non-critical issues detected."
  else
    echo "**Overall: ✅ ALL CHECKS PASSED**"
  fi
  echo ""
  echo "*Heartbeat completed at $(date +%H:%M:%S) in <60s*"
} >> "$LOGFILE"

# --- Touch sentinel for self-monitoring ---
touch /tmp/lab-manager-heartbeat.sentinel

# --- Exit code reflects health ---
if [ "$FAIL" -gt 0 ]; then
  exit 1
else
  exit 0
fi
```

#### Heartbeat vs. Weekly Report

| Aspect | Daily Heartbeat | Weekly Report |
|--------|----------------|---------------|
| **Purpose** | "Is anything broken right now?" | "How is the system performing over time?" |
| **Depth** | Pass/fail per check | Full metrics, trends, recommendations |
| **Runtime** | < 60 seconds | 2-5 minutes |
| **Output** | `heartbeat/YYYY-MM-DD-heartbeat.log` | `YYYY-MM-DD-health-report.md` |
| **Trend data** | No (point-in-time snapshot) | Yes (compares against previous report) |
| **Recommendations** | No (just flags failures) | Yes (evidence-based tuning suggestions) |
| **Schedule** | Daily 07:00 | Sunday 10:00 |

---

## 11. Build Order

### Step 1: Create Directory Structure

```bash
mkdir -p ~/lab-reports
mkdir -p ~/lab-reports/heartbeat    # v1.1
```

### Step 2: Deploy the Lab Manager Agent

Install the SOUL.md from Section 4 into `agents/lab-manager/SOUL.md`.

### Step 3: Create the Lab Manager Breadcrumb

The Lab Manager **must** have a breadcrumb at `agents/lab-manager/agent.lab-manager.crumb.md`. This is required regardless of whether the full Breadcrumb System is deployed — it serves as the Lab Manager's own operational reference and allows other agents to discover it.

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
tags: [operations, health, monitoring, tuning, heartbeat]
deps: [openclaw, jq, bash]
cron: "daily 07:00 (heartbeat), Sunday 10:00 (full report)"
---

# Purpose
Monitor health, effectiveness, and tuning of all deployed OpenClaw infrastructure
specs. Produce daily heartbeat checks and weekly reports with actionable recommendations.

# How It Works
1. Detects which systems are deployed (Memory, Breadcrumbs, Bridge)
2. Reads logs and artifacts from each detected system (read-only)
3. **Daily (07:00):** Runs heartbeat script — pass/fail spot-check of critical metrics
4. **Weekly (Sunday 10:00):** Measures full health, effectiveness, and tuning metrics
5. Compares against previous report for trends (weekly only)
6. Generates structured health report with recommendations (weekly only)
7. Writes outputs to ~/lab-reports/

# Usage
## Commands
\```bash
# Automatic (daily heartbeat — v1.1)
# Runs daily at 07:00
agents/lab-manager/lab-manager-heartbeat.sh

# Automatic (weekly full report)
# Runs Sunday 10:00
lab-manager-run.sh

# Manual trigger
openclaw run lab-manager
\```

## Outputs
| Output | Location | Format | Schedule |
|--------|----------|--------|----------|
| Daily heartbeat | ~/lab-reports/heartbeat/YYYY-MM-DD-heartbeat.log | Markdown/Log | Daily 07:00 |
| Health report | ~/lab-reports/YYYY-MM-DD-health-report.md | Markdown | Sunday 10:00 |
| Run log | ~/lab-reports/lab-manager.log | Log | Every run |

# What It Monitors (v1.1)
- Cron sentinel freshness — detects jobs that stopped running
- Stale .lock files — detects crashed agents that left locks behind
- Correction pipeline health — tracks correction throughput and thrashing
- Standing Directive compliance — indirect/experimental indicators
- All metrics from Memory, Breadcrumb, and Bridge specs

# Gotchas
- 📌 Read-only agent — never modifies files from other systems
- 📌 Adapts to what's deployed — skip metrics for systems that aren't present
- 📌 Weekly runs after all other Sunday maintenance (10:00, after Baker at 08:00)
- 📌 Daily heartbeat is fast (<60s) and non-invasive
- ⚠️ Standing Directive compliance is measured indirectly — accuracy depends
  on quality of CORRECTIONS.md classification and baker.log entries
- ⚠️ Cron sentinel monitoring requires jobs to `touch /tmp/<name>.sentinel` on success
- ⚠️ Lock file detection uses /tmp/*.lock — adjust path if your lock convention differs

# Changelog
- **YYYY-MM-DD (v1.1)**: Added daily heartbeat script, cron sentinel monitoring,
  stale lock detection, correction pipeline health tracking, indirect Standing
  Directive metrics
- **YYYY-MM-DD**: Created
```

### Step 4: Deploy the Daily Heartbeat Script *(v1.1)*

Install the heartbeat script from §10.1:

```bash
# Copy the reference implementation to the agent directory
cp lab-manager-heartbeat.sh agents/lab-manager/lab-manager-heartbeat.sh

# Make it executable
chmod +x agents/lab-manager/lab-manager-heartbeat.sh

# Verify it runs
agents/lab-manager/lab-manager-heartbeat.sh
cat ~/lab-reports/heartbeat/$(date +%Y-%m-%d)-heartbeat.log
```

### Step 5: Schedule the Cron Jobs

Add both the daily heartbeat and weekly report to the system crontab:

```bash
# Lab Manager — daily heartbeat (v1.1)
0 7 * * * /path/to/agents/lab-manager/lab-manager-heartbeat.sh

# Lab Manager — weekly full report
0 10 * * 0 /path/to/lab-manager-run.sh
```

### Step 6: Run Baker

If the Breadcrumb System is deployed, run Baker manually to index the new Lab Manager breadcrumb.

### Step 7: Run Lab Manager Manually

Execute the Lab Manager once to generate the first baseline report. This report will have no trend data (no previous report to compare against) but will establish the baseline for all future comparisons.

Also run the heartbeat script once to verify it produces output:

```bash
agents/lab-manager/lab-manager-heartbeat.sh
```

### Step 8: Verify

- Check `~/lab-reports/` for the generated report file
- Check `~/lab-reports/heartbeat/` for the daily heartbeat log *(v1.1)*
- Open the report and confirm it detected the correct deployed systems
- Confirm metrics are populated (not all UNMEASURABLE)
- Confirm the heartbeat script outputs pass/fail for each check *(v1.1)*
- Verify `agents/lab-manager/agent.lab-manager.crumb.md` exists and has valid frontmatter *(v1.1)*
- If using the messaging channel digest, confirm delivery

---

## Summary

| Aspect | Value |
|--------|-------|
| **Agent count** | 1 |
| **Cron jobs** | 2 (daily 07:00 heartbeat + Sunday 10:00 report) |
| **New file formats** | 0 (uses existing logs and artifacts) |
| **New storage layers** | 0 (one output directory for reports + heartbeat subdirectory) |
| **New extension points** | 0 (consumes, doesn't extend) |
| **Impact on other specs** | None (read-only) |
| **Adapts to deployment** | Yes — measures only what's deployed |
| **Heartbeat runtime** | < 60 seconds |
| **Report time to scan** | < 2 minutes |
| **Required breadcrumb** | `agents/lab-manager/agent.lab-manager.crumb.md` |

---

*This spec adds an operations layer to the OpenClaw infrastructure without increasing the complexity of the systems it monitors. One agent, two cron jobs, one report format. If you remove it, nothing else breaks.*
