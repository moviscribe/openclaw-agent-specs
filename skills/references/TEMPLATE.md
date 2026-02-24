# Breadcrumb Template (v1.1)

Copy and fill in each section:

```markdown
---
id: [type]_[name]
name: [Human-readable name]
type: [script|api|tool|skill|workflow|agent|config]
status: [active|experimental|broken|deprecated]
created: [YYYY-MM-DD]
updated: [YYYY-MM-DD]
owner: [agent-name|shared]
source: [absolute/path/to/component/]
tags: [keyword1, keyword2]
token_estimate: 0
---

# Purpose
[One sentence. What does this component do? If you can't say it in one sentence, you don't understand it yet.]

# How It Works
1. [Step one - literal what happens]
2. [Step two]
3. [Step three]

# Usage
## Commands
```bash
# Real command with actual paths - not placeholders
/absolute/path/to/component/script.sh arg1 arg2
```

## Inputs
| Input | Value / Location | Notes |
|-------|-----------------|-------|
| ARG1 | Command arg | Description |

## Outputs
| Output | Location | Format |
|--------|----------|--------|
| result.json | ~/path/to/output/ | JSON |

# Gotchas
- ⚠️ [Known issue or warning]
- 📌 [Important constraint]
- 🔧 [Workaround if known]

# Related
- [api.related-service.crumb.md](../path/to/api.related-service.crumb.md)

# Changelog
- **YYYY-MM-DD**: Created
- **YYYY-MM-DD**: [Description of change]
```

## Required Fields

| Field | Why |
|-------|-----|
| `# Purpose` | Cold start test — can agent use this? |
| `# Usage` | Copy-paste commands, not placeholders |
| `# Gotchas` | Failure modes, quirks |
| `status` | Determines if Baker indexes it |
| `source` | Path to component (validates existence) |

## Cold Start Test (Mandatory)

Before finishing, ask:

> "Could an agent with zero context use this with ONLY the breadcrumb?"

If NO, add:
- Real commands (not `./script.sh [options]`)
- Actual paths (not `~/path`, use `/Users/bobby/...`)
- Input/output locations

## Token Awareness

Keep breadcrumbs under 500 tokens when possible. Agents load them with context budgets.
