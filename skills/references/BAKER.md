# The Breadcrumb Baker (v1.1)

The Baker is an agent that maintains the Breadcrumb System. It runs periodically to discover, validate, index, and gap-detect breadcrumbs.

## What the Baker Does

### Phase 1: DISCOVER
- Scans workspace directories for undocumented components
- Logs new gaps to `~/.openclaw/gaps/gaps.md`

### Phase 2: VALIDATE
- Checks YAML frontmatter completeness
- Verifies source paths exist
- Calculates token_estimate
- Runs cold start test on each breadcrumb
- Checks recipe.json sync

### Phase 3: DEDUPLICATE + EXTRACT
- Removes duplicate crumbs (keeps newest)
- Extracts capabilities to recipe.json

### Phase 4: INDEX
- Rebuilds `~/.openclaw/breadcrumb-trail/bread-box.json`
- Rebuilds `~/.openclaw/breadcrumb-trail/trail.md`

### Phase 5: GAP DETECTION
- Scans for undocumented components
- Updates gaps.md

### Phase 6: POST-BAKE VERIFICATION
- Logs summary with actionable metrics
- Writes heartbeat-summary.json

## Running the Baker

```bash
# Standard run
~/.openclaw/workspace/agents/breadcrumb-baker/baker.sh

# Check output
cat ~/.openclaw/breadcrumb-trail/bread-box.json | jq '.total_breadcrumbs'
```

## Baker Output

The Baker logs:

```
=== BAKER COMPLETE ===
Total breadcrumbs: 55
Active: 50 | Experimental: 0 | Deprecated: 0 | Broken: 0
Cold start failures: 13
Open gaps: 0
Unowned gaps >48h: 0
Deduplicates removed: 17
```

## Gap States

| State | Meaning | Action |
|-------|---------|--------|
| OPEN | New, unassigned | Assign owner within 48h |
| IN_PROGRESS | Being worked | Update ETA |
| RESOLVED | Fixed | Mark resolved date |
| STALLED | No progress >7d | Escalate |

## Handling Gaps

When you fix a gap:

1. Create/update the breadcrumb
2. Open `~/.openclaw/gaps/gaps.md`
3. Find the gap row
4. Change status to RESOLVED
5. Add resolved date

```markdown
| GAP-001 | script/new-tool.sh | RESOLVED | 2026-02-23 | bobby | - | 2026-02-23 |
```

## Heartbeat Summary

The Baker writes to `~/.openclaw/gaps/heartbeat-summary.json`:

```json
{
  "last_updated": "2026-02-23T21:30:00Z",
  "total_breadcrumbs": 55,
  "active": 50,
  "experimental": 0,
  "cold_start_failures": 13,
  "open_gaps": 0,
  "unowned_gaps_48h": 0
}
```

This is read by the Lab Manager and morning brief.

## Cold Start Validation

The Baker checks:
- Source path exists
- All required sections present
- Commands have real paths

If cold_start fails, it's logged and counted.
