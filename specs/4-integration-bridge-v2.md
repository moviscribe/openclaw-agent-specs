# OpenClaw Integration Bridge — Tri-Domain (Memory × Breadcrumbs × CRM)

*Version: 2.0 | Spec ID: `openclaw-integration-bridge-v2`*
*Audience: OpenClaw operators running two or more primary domain specs.*
*License: Share freely.*

---

## 1. Purpose

The Persistent Memory, Breadcrumb System, and CRM System specs are designed to function independently. This Bridge provides the coordination required to prevent data race conditions, context window overflow, and "identity confusion" when these systems share the same OpenClaw instance.

### Modular "Fail-Soft" Design

* **If CRM is missing**: The Bridge ignores the `crm_tokens` and `crm_entity_exclusion_source` directives; the Context Assembler reallocates that budget to Long-Term Memory automatically.
* **If Breadcrumbs are missing**: The `external_context` slots remain empty, and the system functions as a standard Memory/CRM setup.

---

## 2. Unified Cron Schedule

This schedule ensures that write-intensive tasks (like indexing and mining) do not overlap, preventing filesystem locks.

| Time | Process | System | Phase / Notes |
|:---|:---|:---|:---|
| **02:30** | Soul Keeper | Memory | Identity evolution and SOUL.md updates. |
| **05:00** | Breadcrumb Baker | Breadcrumb | Discover + Validate + Index tools/capabilities. |
| **06:00** | Memory Miner | Memory | Extract chronological memories from the previous day. |
| **06:30** | CRM Miner | CRM | **(Optional)** Update relationship/entity profiles. |
| **07:00** | Conflict Detector | Memory | Cross-source consistency check across all logs. |
| **11:00** | Breadcrumb Baker | Breadcrumb | Standard mid-day tool reindex. |
| **17:00** | Breadcrumb Baker | Breadcrumb | Standard afternoon tool reindex. |
| **23:00** | Breadcrumb Baker | Breadcrumb | Final daily tool reindex. |

---

## 3. Modular CONFIG.md Architecture

Add these values to your central `CONFIG.md`. The logic is designed to be "null-safe"—if a path does not exist, the specific integration hook remains inert.

```yaml
# ══════════════════════════════════════════════════════════════
# Integration Bridge v2.0 - Token Budgets & Routing
# ══════════════════════════════════════════════════════════════
context_budget_tokens: 6000

# Primary Domain Budgets
identity_tokens: 800
user_profile_tokens: 600
corrections_tokens: 400
crm_tokens: 1000                # Set to 0 if CRM spec is not used
external_context_budget: 800    # Reserved for Breadcrumbs

# Secondary/Recall Budgets
recent_memory_tokens: 1100
semantic_recall_tokens: 1000
longterm_memory_tokens: 300     # Absorbs CRM/External budget if null

# Integration Hooks
external_context_source: "~/breadcrumb-trail/recipe.json"
recall_external_handler: "~/breadcrumb-trail/scripts/baker-reindex.sh"
crm_entity_exclusion_source: "~/breadcrumb-trail/recipe.json"

heartbeat_custom_patterns:
  - name: "breadcrumb_triggers"
    patterns:
      - '/broke|broken|stopped working|not working/i'
      - '/(there.s|we have|we built|we made|i made) (already |)(a |an |)(script|tool|workflow|bot|agent|dashboard|pipeline)/i'
      - '/build me a/i'
      - '/what (tools|scripts|capabilities|integrations) do (we|you|i) have/i'
    route_to: "breadcrumb-directive"
    priority: "normal"
```

---

## 4. Context Assembler — Loading Priority

When the agent initializes, the Context Assembler loads data in this specific order to ensure the most "stable" facts (Identity and Tools) are never truncated by "volatile" facts (Recent Memories):

1. **Identity (`SOUL.md`)**: Who the agent is.
2. **Operator Baseline (`USER.md`)**: Who they are talking to.
3. **Capability Map (`recipe.json`)**: What tools are available (Breadcrumb summary).
4. **CRM Profiles**: Who/What is involved in the current task (if CRM exists).
5. **Recent Corrections**: Immediate feedback from the operator.
6. **Recent Memory**: The most immediate conversational context.
7. **Semantic Recall**: Relevant past experiences.
8. **Long-Term Memory**: Deep historical context.

---

## 5. Recall Failure Routing

The Recall Validator uses a three-way branch to handle corrections. If an agent "forgets" something, it classifies the failure to ensure the right database is updated:

* **Type: Tool / Capability** → Route to `baker-reindex.sh`. This triggers Baker to see if a tool is documented but missing from the index.
* **Type: Relationship / Entity** → Route to the CRM Archive. Updates the specific Markdown profile for that person or place.
* **Type: Chronological / Knowledge** → Route to the corrections log and `MEMORY.md`. Standard chronological memory update.

---

## 6. CRM Entity Exclusion Protocol

To prevent "Profile Bloat," the CRM Miner must check the Breadcrumb system before creating a new entity profile.

### The Logic

1. Parse the operator mention.
2. Check `recipe.json` capability keys and summary strings for matches.
3. **If matched**: Skip full profile creation. Append a cross-reference note into the operator's baseline profile (e.g., *"Uses [tool] — see [breadcrumb_id] for integration details"*).
4. **If NOT matched**: Proceed with standard CRM entity profile creation.

---

## 7. Error Recovery & Unified Fallbacks

If any single system fails, the Bridge provides a "graceful degradation" path handled natively by the Context Assembler:

| Failed Resource | Unified Fallback Sequence | System |
|:---|:---|:---|
| Agent Identity | Latest snapshot from `SOUL_HISTORY/`. | Memory |
| Vector DB | Keyword search against chronological memory and CRM archive. | Memory/CRM |
| Capabilities Manifest | Filesystem scan for `*.crumb.md` extensions. | Breadcrumb |
| CRM Archive | Skip slot; reallocate CRM tokens to Long-Term Memory. | CRM |

---

## 8. Build Order

1. **Verify Individual Deployments**: Confirm Memory, Breadcrumbs, and (optionally) CRM are fully operational in isolation.
2. **Apply Configuration**: Update your master `CONFIG.md` with the integrated token budgets and routing parameters from Section 3.
3. **Deploy Scripts**: Create the `baker-reindex.sh` script for the recall hook and apply execution permissions.
4. **Update Cron**: Align all system schedule entries to the Unified Schedule in Section 2.
5. **Cross-Register Agents**: Create `.crumb.md` files for the `memory-miner`, `crm-miner`, `soul-keeper`, etc., so they appear in the tool index.
6. **Trigger Reindex**: Run the Baker agent manually once to index the new agent breadcrumbs.
