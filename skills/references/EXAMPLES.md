# Examples (v1.1)

## ✅ GOOD — Passes Cold Start Test

### Script Example

```markdown
---
id: script_add-media
name: Add Media (Overseerr)
type: script
status: active
created: 2026-02-23
updated: 2026-02-23
owner: bobby
source: /Users/bobby/.openclaw/workspace/agents/family-assistant/
tags: [media, plex, overseerr]
token_estimate: 350
---

# Purpose
Add movies or TV shows to Plex via Overseerr API.

# How It Works
1. User provides title and type (movie/tv)
2. Script searches Overseerr for the title
3. Gets TMDB ID from search results
4. POSTs request to Overseerr API

# Usage
## Commands
```bash
/Users/bobby/.openclaw/workspace/agents/family-assistant/scripts/add-media.sh movie "Dune"
```

## Inputs
| Input | Value | Notes |
|-------|-------|-------|
| Media type | Command arg | movie, tv, or show |
| Title | Command arg | Full title in quotes |

## Outputs
| Output | Location | Format |
|--------|----------|--------|
| API response | stdout | JSON with request details |

# Gotchas
- ⚠️ Requires Overseerr API key in .secrets/
- 📌 Must be on local network to reach Blacktower
- 🔧 Key location: ~/.secrets/overseerr-api-key.txt

# Related
- [tool.plex.crumb.md](../tools/plex.tool.crumb.md)

# Changelog
- **2026-02-23**: Created
```

---

### Tool Example

```markdown
---
id: tool_home-assistant
name: Home Assistant
type: tool
status: active
created: 2026-02-23
updated: 2026-02-23
owner: bobby
source: http://192.168.50.90:8123
tags: [smart-home, iot]
token_estimate: 450
---

# Purpose
Control smart home devices (lights, climate, switches) via Home Assistant API.

# Usage
## Commands
```bash
curl -s -X POST -H "Authorization: Bearer $HA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entity_id": "light.johns_light"}' \
  http://192.168.50.90:8123/api/services/light/turn_on
```

## Inputs
| Input | Value | Notes |
|-------|-------|-------|
| Entity ID | User request | e.g., light.johns_light |
| Action | User request | turn_on, turn_off |

## Outputs
| Output | Location | Format |
|--------|----------|--------|
| HA API response | stdout | JSON |

# Gotchas
- ⚠️ Must be on local network (VPN or home)
- 📌 Token expires — refresh via HA UI
- 🔧 Get token: Profile → Long-Lived Access Token

# Changelog
- **2026-02-23**: Created
```

---

## ❌ BAD — Fails Cold Start Test

### Example 1: Placeholder Commands

```markdown
# Purpose
Media addition tool

# Usage
./add-media.sh [options]
```

**Why it fails:** No real paths, no actual command format.
**Fix:** `/Users/bobby/.openclaw/workspace/.../add-media.sh movie "Title"`

---

### Example 2: No Purpose

```markdown
# How It Works
1. User runs script
2. Script processes
3. Returns output
```

**Why it fails:** No # Purpose section.
**Fix:** Add "Add media to Plex via Overseerr" as one sentence.

---

### Example 3: Missing Gotchas

```markdown
# Purpose
Control lights via HA

# Usage
curl ...turn_on
```

**Why it fails:** No warning about network or token expiry.
**Fix:** Add Gotchas section with ⚠️ network, 📌 token expiry.

---

### Example 4: Status Missing

```markdown
---
id: script_foo
name: Foo Script
# status: missing!
source: /path/
---
```

**Why it fails:** Baker won't index without status.
**Fix:** Add `status: active`

---

### Example 5: Placeholder Args

```markdown
# Usage
./process.sh --input <file> --output <dir>
```

**Why it fails:** Angle brackets are placeholders.
**Fix:** `./process.sh --input /data/input.json --output /data/results/`

---

## Quick Checklist

Before finishing a breadcrumb, verify:

- [ ] Has `# Purpose` (one sentence)
- [ ] Has `# Usage` with REAL commands (not [options])
- [ ] Has `# Gotchas` with warnings
- [ ] YAML has `status: active|experimental`
- [ ] YAML has `source:` with REAL absolute path
- [ ] Commands use absolute paths
- [ ] Passes cold start test: "Could I use this with zero context?"
