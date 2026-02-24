---
name: breadcrumb-creator
description: Create and maintain breadcrumbs — structured documentation. v1.1 adds decision tree, cold start test, and checklist.
metadata: {"openclaw": {"emoji": "🍞", "requires": {"bins": ["jq", "bash"]}, "author": "bobby", "version": "1.1"}}
allowed-tools: Read Write Exec Bash
---

# Breadcrumb Creator (v1.1)

This skill guides you through creating and maintaining breadcrumbs.

## When to Use This Skill

Use when:
- Building a new script, tool, API, agent, workflow, or skill
- Fixing a gap logged by the Baker
- Modifying an existing component
- Performing the cold start test

## The Decision Tree

```
Does the component already have a breadcrumb?
├── YES → Is it accurate?
│   ├── YES → Done
│   └── NO → Update it (Purpose, Usage, Gotchas)
└── NO → Create one
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

Before finishing, ask:

> "Could an agent with zero context use this with ONLY the breadcrumb?"

If NO, fix:
- Real commands (not `./script.sh [options]`)
- Real paths (not `~/path`, use `/Users/bobby/...`)

## Quick Reference

| Question | Answer |
|----------|--------|
| Where to put? | Component's directory (in-situ) |
| bread-box.json | `~/.openclaw/breadcrumb-trail/bread-box.json` |
| recipe.json | `~/.openclaw/breadcrumb-trail/recipe.json` |
| Gaps? | `~/.openclaw/gaps/gaps.md` |
| Run Baker? | `~/.openclaw/workspace/agents/breadcrumb-baker/baker.sh` |

## Common Mistakes

1. Placeholder commands
2. No purpose
3. Missing gotchas
4. Wrong status
5. Source path wrong

## Post-Creation Checklist

- [ ] Has `# Purpose` (one sentence)
- [ ] Has `# Usage` with real commands
- [ ] Has `# Gotchas`
- [ ] YAML has `status: active`
- [ ] YAML has `source:` with real path
- [ ] Passes cold start test

## See Also

- [Template](references/TEMPLATE.md)
- [Conventions](references/CONVENTIONS.md)
- [Examples](references/EXAMPLES.md)
- [Baker](references/BAKER.md)
