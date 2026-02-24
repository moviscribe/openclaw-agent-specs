# QMD Local Search Engine for OpenClaw

*Version: 1.0 | Spec ID: `openclaw-qmd-integration`*
*Audience: OpenClaw operators (Bobby, Rob) wanting local document search*
*License: Share freely. Customize per deployment.*
*Changelog: [See Section 12](#12-changelog)*

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Prerequisites](#2-prerequisites)
3. [Installation](#3-installation)
4. [Configuration](#4-configuration)
5. [Collections Setup](#5-collections-setup)
6. [Context Metadata](#6-context-metadata)
7. [MCP Server](#7-mcp-server)
8. [CLI Tools](#8-cli-tools)
9. [Agent Integration](#9-agent-integration)
10. [Startup Integration](#10-startup-integration)
11. [Troubleshooting](#11-troubleshooting)
12. [Changelog](#12-changelog)

---

## 1. Purpose

This spec defines how to install and configure QMD — a local search engine combining BM25 keyword search, vector semantic search, and LLM reranking — for OpenClaw agent deployments.

### What This Spec Delivers

- Local search across all workspace markdown documents
- Semantic search capability using GGUF models (runs locally, no API calls)
- MCP server for agent integration
- Startup integration for contextual awareness
- Collections for different document types

### What This Spec Does Not Cover

- Model fine-tuning — QMD handles this automatically
- Network deployment — QMD is designed for local use
- Authentication — MCP runs on localhost only

---

## 2. Prerequisites

### Hardware
- macOS or Linux (tested on macOS 25.2.0, Ubuntu)
- 4GB+ RAM recommended
- Apple M4 Metal GPU (macOS) or CUDA (Linux) for embeddings

### Software
- Node.js 18+ or Bun runtime
- npm or bun package manager

### Access
- Write access to `~/.openclaw/workspace/`
- Write access to `~/.local/bin/` (for CLI wrappers)

---

## 3. Installation

### Step 1: Install QMD globally

```bash
npm install -g @tobilu/qmd
```

Or with bun:
```bash
bun install -g @tobilu/qmd
```

Verify:
```bash
qmd --help
```

### Step 2: Create workspace collection

```bash
qmd collection add ~/.openclaw/workspace --name workspace
```

This indexes all `.md` files in the workspace. On first run, it creates embeddings (~3-5 minutes for 1500 files).

### Step 3: Create bobworld collection (optional but recommended)

```bash
qmd collection add ~/.openclaw/workspace/bobworld --name bobworld
```

---

## 4. Configuration

### Collections Directory

Create directories for different document types:

| Collection | Path | Purpose |
|------------|------|---------|
| workspace | ~/.openclaw/workspace/ | All agent docs, memory, skills |
| bobworld | ~/.openclaw/workspace/bobworld/ | Dashboard code |
| archive | /Volumes/bobWorld/Archive/ | Long-term memory (if NFS available) |

### Check Status

```bash
qmd status
```

Output:
```
QMD Status

Index: /Users/bobby/.cache/qmd/index.sqlite
Size:  35.7 MB
MCP:   running (PID 34622)

Collections
  workspace (qmd://workspace/)
    Pattern:  **/*.md
    Files:    1495
    Contexts: 4
```

---

## 5. Collections Setup

### Adding a Collection

```bash
qmd collection add /path/to/docs --name mycollection
```

Options:
- `--mask "*.md"` — File pattern (default: **/*.md)
- `--name` — Collection name (required)

### Removing a Collection

```bash
qmd collection remove mycollection
```

### Re-indexing

```bash
qmd update              # Re-index all collections
qmd update --pull       # Git pull first, then re-index
```

### Generating Embeddings

Embeddings enable semantic (vector) search:

```bash
qmd embed
```

This downloads GGUF models (~1.3GB) and creates vector embeddings for all documents. Takes 3-5 minutes for 1500 files.

---

## 6. Context Metadata

Context helps QMD understand what each collection contains, improving search results.

### Add Context

```bash
qmd context add qmd://workspace/docs "Technical documentation, specs, and architecture decisions"
qmd context add qmd://workspace/memory "Daily logs of agent activities, decisions, and learnings"
qmd context add qmd://workspace/agents "Agent configurations, SOUL.md files, and run scripts"
qmd context add qmd://workspace/skills "OpenClaw skills for various integrations"
```

### List Contexts

```bash
qmd context list
```

### Remove Context

```bash
qmd context rm qmd://workspace/docs
```

---

## 7. MCP Server

QMD exposes an MCP server for agent integration.

### Start MCP Server

```bash
# As daemon (recommended for persistent use)
qmd mcp --http --daemon

# Or foreground
qmd mcp --http
```

Default port: 8181
URL: `http://localhost:8181/mcp`

### Stop MCP Server

```bash
qmd mcp stop
```

### Tools Exposed

| Tool | Description |
|------|-------------|
| `qmd_search` | BM25 keyword search |
| `qmd_vector_search` | Semantic vector search |
| `qmd_deep_search` | Hybrid + reranking |
| `qmd_get` | Get document by path/docid |
| `qmd_multi_get` | Get multiple documents |
| `qmd_status` | Index health |

### Logs

Logs stored at: `~/.cache/qmd/mcp.log`

---

## 8. CLI Tools

### Search Commands

```bash
# Fast keyword search (BM25)
qmd search "cron jobs"

# Semantic vector search (requires embeddings)
qmd vsearch "John family preferences"

# Hybrid search with reranking (best quality, requires model)
qmd query "automations dashboard"
```

### Get Documents

```bash
# Get document content
qmd get workspace/docs/spec-agent-creation.md

# Get with line numbers
qmd get workspace/docs/spec-agent-creation.md --line-numbers

# Get specific lines
qmd get workspace/docs/spec-agent-creation.md:20 -l 10
```

### Output Formats

```bash
# JSON output (good for agents)
qmd search "query" --json -n 10

# File paths only
qmd search "query" --files -n 20

# Markdown
qmd search "query" --md
```

### Collection Filter

```bash
# Search only in workspace
qmd search "query" -c workspace

# Search only in bobworld
qmd search "query" -c bobworld
```

---

## 9. Agent Integration

### CLI Wrapper Script

Create `~/.local/bin/qmd-search`:

```bash
#!/bin/bash
# QMD search wrapper for agents
query="$1"
format="${2:-text}"

if [ -z "$query" ]; then
  echo "Usage: qmd-search \"query\" [text|json|files]"
  exit 1
fi

case "$format" in
  json) qmd search "$query" --json -n 10 ;;
  files) qmd search "$query" --files -n 20 ;;
  *) qmd search "$query" -n 5 ;;
esac
```

Make executable:
```bash
chmod +x ~/.local/bin/qmd-search
```

### Usage

```bash
# Text output (default)
qmd-search "cron jobs"

# JSON for parsing
qmd-search "automations" json

# File paths only
qmd-search "spec" files
```

### Creating a Skill

Create `~/.openclaw/workspace/skills/qmd/SKILL.md`:

```markdown
# QMD - Local Search Engine

Search local markdown documents using QMD (BM25 + Vector + LLM reranking).

## Commands

- `qmd search "query"` - Fast keyword search (BM25)
- `qmd vsearch "query"` - Semantic vector search
- `qmd query "query"` - Hybrid search with reranking
- `qmd get "path"` - Get document content
- `qmd status` - Show index status

## Examples

qmd search "cron jobs"
qmd vsearch "John family preferences"
qmd get "workspace/docs/spec-agent-creation.md"

## Collections

- workspace: ~/.openclaw/workspace/
- bobworld: ~/.openclaw/workspace/bobworld/
```

---

## 10. Startup Integration

### Update AGENTS.md

Add QMD query to session startup in `~/.openclaw/workspace/AGENTS.md`:

```markdown
## Every Session

Before doing anything else:

1. Read `SOUL.md` — this is who you are
2. Read `USER.md` — this is who you're helping
3. Read **last 2 days only** of `memory/YYYY-MM-DD.md` (today + yesterday) for recent context
4. **Query QMD** — Run `qmd-search "recent project topic"` to find relevant docs
5. **If in MAIN SESSION** (direct chat with your human): 
   - Read `MEMORY.md` — long-term curated memory
...
```

### Update Fallback Search Order

```markdown
**CRITICAL:** If John asks about something you "built together" but you don't remember:
1. Check **QMD** first: `qmd-search "KEYWORD"` (fast local search)
2. Check Vector KB: `bash scripts/query-vector-kb.sh "KEYWORD"`
3. Check `MEMORY.md` and `memory/` files
4. Check WikiJS: http://192.168.50.90:8091
5. If still unknown, spawn the **Systems Auditor** to verify against docs
```

### Add to TOOLS.md

```markdown
## QMD Local Search
- **What:** Local search engine for markdown docs (BM25 + Vector + LLM reranking)
- **Collections:** workspace (~1500 files), bobworld (74 files)
- **MCP Server:** http://localhost:8181/mcp
- **CLI Wrapper:** `~/.local/bin/qmd-search`
- **Usage:**
  - `qmd search "query"` - Fast BM25
  - `qmd vsearch "query"` - Semantic vectors
  - `qmd query "query"` - Hybrid with reranking
  - `qmd get "path"` - Get document
  - `qmd-search "query" files` - Agent-friendly output
```

---

## 11. Troubleshooting

### Embeddings not working

```bash
# Re-generate embeddings
qmd embed -f   # -f forces regeneration
```

### MCP server not responding

```bash
# Check if running
qmd status

# Restart
qmd mcp stop
qmd mcp --http --daemon
```

### No search results

- Check collections exist: `qmd status`
- Verify files indexed: `qmd ls workspace`
- Re-index: `qmd update`

### NFS stale handle (Archive collection)

If `/Volumes/bobWorld/Archive/` gives "stale NFS file handle":
- Unmount and remount: `umount /Volumes/bobWorld && mount /Volumes/bobWorld`
- Or restart Unraid server

### Model download slow

QMD downloads GGUF models on first vector search. Check:
- Internet connection
- Disk space (needs ~2GB for models)
- GPU availability

---

## 12. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-22 | Initial spec — installation, collections, MCP, CLI, startup integration |
