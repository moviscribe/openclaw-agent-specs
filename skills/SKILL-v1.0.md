---
name: breadcrumb-creator
description: Create and maintain breadcrumbs for scripts, tools, APIs, agents, workflows, and skills. Use when building new components, fixing gaps, updating documentation, or modifying existing components.
metadata: {"openclaw": {"emoji": "🍞", "requires": {"bins": ["jq", "bash"]}, "author": "rob", "version": "1.0"}}
allowed-tools: Read Write Exec Bash
---

# Breadcrumb Creator

This skill guides you through creating and maintaining breadcrumbs — structured documentation for any component.

## When to Use This Skill

Use this skill when:
- Building a new script, tool, API, agent, workflow, or skill
- Fixing a gap logged by the Baker
- Modifying an existing component
- Performing the "cold start test" on documentation

## Quick Reference

| Question | Answer |
|----------|--------|
| Where to put breadcrumbs? | In component's directory (in-situ) |
| Where is recipe.json? | `~/breadcrumb-trail/recipe.json` |
| Where are gaps? | `~/.openclaw/gaps/gaps.md` |
| How to run Baker? | `/home/rob/.openclaw/scripts/breadcrumb-baker.sh` |

## The Decision Tree

```
Does the component already have a breadcrumb?
├── YES → Is it accurate?
│   ├── YES → Done
│   └── NO → Update it (Purpose, Usage, Gotchas)
└── NO → Create one (see references/)
```

## Required Sections

Every breadcrumb MUST have:

1. **# Purpose** — One sentence. What does this component do?
2. **# Usage** — Real copy-paste commands, not placeholders
3. **# Gotchas** — Failure modes, quirks, warnings
4. **YAML frontmatter** — id, name, type, status, source

## See Also

- [Template](references/TEMPLATE.md) — Full breadcrumb template
- [Naming Conventions](references/CONVENTIONS.md) — ID and filename rules
- [Examples](references/EXAMPLES.md) — Good vs bad breadcrumbs
- [Baker Documentation](references/BAKER.md) — How the Baker works
