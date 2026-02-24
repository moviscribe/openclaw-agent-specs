# Naming Conventions (v1.1)

## Breadcrumb ID Format

Format: `{type}_{component-name}`

- Underscores between type and name
- Hyphens within the component name
- All lowercase

| Type | ID Example | Filename |
|------|-----------|----------|
| Script | `script_hello-world` | `script.hello-world.crumb.md` |
| API | `api_gmail-access` | `api.gmail-access.crumb.md` |
| Tool | `tool_dashboard` | `tool.dashboard.crumb.md` |
| Skill | `skill_weather` | `skill.weather.crumb.md` |
| Workflow | `workflow_data-pipeline` | `workflow.data-pipeline.crumb.md` |
| Agent | `agent_assistant` | `agent.assistant.crumb.md` |
| Config | `config_cron-jobs` | `config.cron-jobs.crumb.md` |

## Directory Structure

### In-Situ (Source of Truth)

For components you built:

```
~/scripts/my-script/
├── run.sh
└── script.my-script.crumb.md

~/agents/my-agent/
├── main.py
└── agent.my-agent.crumb.md
```

### Orphans

For external APIs or components without a local directory:

```
~/breadcrumb-trail/orphans/
├── api.gmail-access.crumb.md
└── config.cron-jobs.crumb.md
```

## YAML Frontmatter Required Fields

```yaml
id: script_example          # Must match filename (minus .crumb.md)
name: Example Script        # Human-readable name
type: script               # script|api|tool|skill|workflow|agent|config
status: active            # active|experimental|broken|deprecated
created: 2026-02-23       # YYYY-MM-DD
updated: 2026-02-23       # YYYY-MM-DD (update when modifying)
owner: bobby               # Agent name or shared
source: /path/to/component/  # Absolute path to component
```

## Optional Fields

```yaml
tags: [keyword1, keyword2]  # For search
deps: [python3, jq]        # Dependencies
cron: "0 6 * * *"         # If scheduled
```

## Common Mistakes (v1.1)

1. **Wrong ID format** — `script/my-script` → `script_my-script`
2. **Duplicate crumbs** — In both tool dir AND orphans → keep in-situ only
3. **Missing status** — Defaults to broken → set `active` or `experimental`
4. **Source path wrong** — `~/path` → `/Users/bobby/.openclaw/path`
5. **No commands** — Just describing → Write actual commands
6. **Placeholder args** — `./run.sh [options]` → `./run.sh --flag value`
7. **No gotchas** — Expects perfection → Add failure modes
8. **Token bloat** — 2000+ tokens → Trim to essentials
