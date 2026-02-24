---
id: skill_breadcrumb-creator
name: Breadcrumb Creator
type: skill
status: active
created: 2026-02-23
updated: 2026-02-23
owner: bobby
source: /Volumes/bobWorld/Knowledge Sharing/Bob-Custom-Skills/breadcrumb-creator/
tags: [breadcrumbs, documentation, skill, creation, enforcement]
token_estimate: 600
---

# Purpose
Skill that guides agents to create and maintain breadcrumbs with ENFORCEMENT. Contains MUST rules from spec Section 18.

# How It Works
1. Agent loads skill when task involves documentation
2. Skill provides MANDATORY RULES (MUST comply)
3. Agent follows decision tree: build receipt → build → crumb → register
4. Cold start validation required before registration
5. Remediation queue checked at session start

# Mandatory Rules (MUST)

| Rule | What | Failure |
|------|------|---------|
| Build Receipt | Create `.build-in-progress` at build start | HIGH severity gap |
| Register | Run `register-crumb.sh` after crumb | Remediation task |
| Cold Start | Pass `cold-start-check.sh` | Registration aborts |
| Modify | Update `updated` date + changelog | Stale crumb |
| Break | Add to Gotchas + set `broken` | None |
| Remediation | Check queue at session start | Tasks pile up |

# Usage
```bash
# Validate crumb before registration
~/.openclaw/scripts/cold-start-check.sh /path/to/crumb.crumb.md

# Register crumb (validates + indexes + updates recipe)
~/.openclaw/scripts/register-crumb.sh /path/to/crumb.crumb.md

# Check remediation queue
ls ~/.openclaw/remediation/bobby/
```

# Trigger Conditions
- Building new script/tool/agent/workflow/skill
- Fixing a gap
- Modifying existing component

# Structure
- SKILL.md — Summary + MANDATORY RULES + decision tree
- references/
  - TEMPLATE.md — Full breadcrumb template
  - CONVENTIONS.md — Naming rules
  - EXAMPLES.md — Good vs bad breadcrumbs
  - BAKER.md — How the Baker works

# Key Additions (v1.2)
- MANDATORY RULES section (from spec Section 18)
- Build receipt requirement
- Registration hook requirement
- Remediation queue check
- Failure consequences documented

# Related
- [spec.breadcrumb-v3.6.crumb.md](../Openclaw%20Spec%20Docs/2_-_OpenClaw_Breadcrumb_System_Spec_v3_6.md)

# Changelog
- **2026-02-23 (v1.2)**: Added MANDATORY RULES - enforcement language from spec
- **2026-02-23 (v1.1)**: Added decision tree, cold start test, checklist
- **2026-02-23**: Created from Rob's v2
