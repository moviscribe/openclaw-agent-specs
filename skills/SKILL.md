---
name: breadcrumb-creator
description: Create and maintain breadcrumbs — structured documentation for scripts, tools, APIs, agents, workflows, and skills. Use when building new components, fixing gaps, updating documentation, or modifying existing components.
metadata: {"openclaw": {"emoji": "🍞", "requires": {"bins": ["jq", "bash"]}, "author": "bobby", "version": "1.2"}}
allowed-tools: Read Write Exec Bash
---

# Breadcrumb Creator (v1.2)

This skill guides you through creating and maintaining breadcrumbs — the living map that lets any agent (including future-you with amnesia) use your components.

## When to Use This Skill

Use when:
- Building a new script, tool, API, agent, workflow, or skill
- Fixing a gap logged by the Baker
- Modifying an existing component
- Performing the cold start test

---

# MANDATORY RULES (MUST)

These rules are ENFORCED by the system. Non-compliance generates remediation tasks.

## RULE 1: Build Receipt (MUST)

When you start building a NEW component:
```
echo '{"agent": "bobby", "started": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'", "task": "description", "expected_crumb": "script.name.crumb.md"}' > /path/to/component/.build-in-progress
```

## RULE 2: Create Crumb (MUST)

When you finish building:
- Create `.crumb.md` in component's directory (in-situ)
- Must pass `cold-start-check.sh` before registration

## RULE 3: Register Immediately (MUST)

After creating crumb, MUST run:
```
~/.openclaw/scripts/register-crumb.sh /path/to/component/script.name.crumb.md
```

This validates, adds to bread-box, updates recipe, removes build receipt.

## RULE 4: Cold Start Validation (MUST)

Before finalizing any crumb, it MUST pass:
```
~/.openclaw/scripts/cold-start-check.sh /path/to/crumb.crumb.md
```

If it fails — fix errors and re-run. Registration aborts on failure.

## RULE 5: Modify Existing (MUST)

When modifying a Component:
- Update crumb's `updated` date
- Add changelog entry
- SHOULD re-run `register-crumb.sh`

## RULE 6: When It Breaks (MUST)

- Add failure to `# Gotchas` section
- Set status to `broken` if can't fix

## RULE 7: Remediation Queue (MUST)

At session start, check:
```
ls ~/.openclaw/remediation/bobby/*.json
```

Process OPEN tasks BEFORE starting new work. Set status to RESOLVED when complete.

---

# The Decision Tree

```
Does the component already have a breadcrumb?
├── YES → Is it accurate?
│   ├── YES → Done
│   └── NO → Update it (Purpose, Usage, Gotchas)
└── NO → Create one (MUST: build receipt → build → crumb → register)
```

## Required Sections

Every breadcrumb MUST have:

| Section | Why |
|---------|-----|
| `# Purpose` | One sentence. If you can't say it, you don't understand it. |
| `# Usage` | Real copy-paste commands with actual paths — not TODO |
| `# Gotchas` | Failure modes, quirks, warnings |
| YAML frontmatter | id, name, type, status, source |

## Cold Start Test

Before finishing ANY breadcrumb:

> "Could an agent with ZERO context use this with ONLY the breadcrumb?"

---

# Quick Reference

| Question | Answer |
|----------|--------|
| Where to put? | Component's directory (in-situ) |
| bread-box.json | `~/.openclaw/breadcrumb-trail/bread-box.json` |
| recipe.json | `~/.openclaw/breadcrumb-trail/recipe.json` |
| Gaps? | `~/.openclaw/gaps/gaps.md` |
| Remediation? | `~/.openclaw/remediation/bobby/` |
| Register? | `~/.openclaw/scripts/register-crumb.sh` |
| Validate? | `~/.openclaw/scripts/cold-start-check.sh` |
| Baker? | `~/.openclaw/workspace/agents/breadcrumb-baker/baker.sh` |

## Common Mistakes

1. **Skip build receipt** → HIGH severity gap
2. **Skip registration** → Remediation task generated
3. **Placeholder commands** → Cold start fails
4. **No purpose** → Cold start fails
5. **Missing gotchas** → Cold start fails
6. **Wrong status** → Cold start fails
7. **Source path wrong** → Cold start fails
8. **Don't check remediation** → Unresolved tasks accumulate

## Post-Creation Checklist

Before finishing:
- [ ] Build receipt created (`.build-in-progress`)
- [ ] `# Purpose` is one sentence
- [ ] `# Usage` has real commands with real paths
- [ ] `# Gotchas` has at least one warning
- [ ] YAML has `status: active`
- [ ] YAML has `source:` with real path
- [ ] Cold start check passes (`cold-start-check.sh`)
- [ ] Registered (`register-crumb.sh`)

---

## See Also

- [Template](references/TEMPLATE.md) — Full breadcrumb template
- [Conventions](references/CONVENTIONS.md) — Naming rules
- [Examples](references/EXAMPLES.md) — Good vs bad breadcrumbs
- [Baker](references/BAKER.md) — How the Baker works

---

## Changelog

- **v1.2**: Added MANDATORY RULES (Section 18 from spec) — build receipt, registration, remediation queue
- **v1.1**: Added decision tree, cold start test emphasis, checklist, token awareness
- **v1.0**: Initial version (Rob)
