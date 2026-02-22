# openclaw-agent-specs

> **Your AI agent forgets everything every session. These specs fix that.**

You spin up an agent. It's brilliant. It writes code, manages tasks, holds a great conversation. Then the session ends. Tomorrow it wakes up with no idea who you are, what it built yesterday, or what tools it has access to. It's digital amnesia on repeat — and if you've run agents in production for more than a week, you know exactly how painful this gets.

We got tired of babysitting. So we wrote the specs to make agents that actually *persist*.

---

## What This Is

A collection of **battle-tested specification documents** for building persistent, self-aware AI agent systems. Born from running agents 24/7 on [OpenClaw](https://github.com/nickhobbs94/openclaw) and solving every "why did it forget?" problem the hard way.

These specs give your agent:
- **Memory that survives restarts** — not just chat history, real structured memory
- **Self-awareness of its own capabilities** — no more "I don't have access to that" when it does
- **Relationship continuity** — it remembers who people are and what matters to them
- **Operational health monitoring** — so you know when things drift before they break

The patterns are designed for OpenClaw but are **framework-agnostic**. If you're building on LangChain, AutoGPT, CrewAI, or a custom setup, the architecture translates.

---

## The Problem

Every agent framework gives you tool-calling and chat. Almost none of them solve the *operational* problems that show up when you actually run agents continuously:

| Problem | What It Looks Like |
|---|---|
| **Session Amnesia** | Agent has zero memory between sessions. Every conversation starts from scratch. |
| **Capability Blindness** | Agent doesn't know what tools, scripts, or integrations are available to it. You end up re-explaining its own setup constantly. |
| **Relationship Amnesia** | Agent forgets who people are, what they care about, and the context of past interactions. |
| **No Self-Improvement Loop** | Agent makes the same mistakes repeatedly. No mechanism to learn from failures or evolve behavior over time. |
| **No Operational Monitoring** | You have no idea if memory is bloating, recall is degrading, or integrations are silently failing — until something breaks visibly. |

These aren't edge cases. They're the *default experience* of running any agent for more than a day.

---

## The Specs

Seven specifications, each solving a distinct problem, designed to compose together:

| Spec | Version | Description |
|:-----|:--------|:------------|
| **0 — Agent Creation** | v3 | How to properly create, register, and manage agent lifecycles with async telemetry. The foundation everything else builds on. |
| **1 — Persistent Memory** | v4 / v4.1 | Five-agent memory system: mining, identity evolution, recall validation, conflict detection, and compaction. This is the core. |
| **2 — Breadcrumb System** | v3 / v3.1 | Self-documenting component registry so agents can discover their own capabilities at runtime. No more capability blindness. |
| **3 — CRM System** | v2 / v2.1 | Relationship intelligence — agents remember people, context, preferences, and connections between individuals. |
| **4 — Integration Bridge** | v2 / v2.1 | Coordination layer connecting Memory, Breadcrumbs, and CRM without tight coupling. The glue that keeps systems composable. |
| **5 — Deployment Auditor** | v1 | Automated verification that all specs are correctly implemented. Catches drift and misconfiguration before they cause problems. |
| **6 — Lab Manager** | v1 / v1.1 | Operational health monitoring and tuning recommendations. Keeps everything running cleanly over time. |

---

## Architecture

Here's how the specs connect:

```
┌─────────────────────────────────────────────────────┐
│                   Lab Manager (6)                    │
│              monitors health of everything           │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────┼──────────────────────────────┐
│                      │        Deployment Auditor (5) │
│                      │     verifies all specs        │
│  ┌───────────────────┼───────────────────────┐      │
│  │                   │                       │      │
│  │    ┌──────────────┴──────────────┐        │      │
│  │    │    Integration Bridge (4)    │        │      │
│  │    │    loose coupling layer      │        │      │
│  │    └──┬───────────┬──────────┬───┘        │      │
│  │       │           │          │             │      │
│  │       ▼           ▼          ▼             │      │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐     │      │
│  │  │ Memory  │ │Breadcrumb│ │   CRM   │     │      │
│  │  │  (1)    │ │   (2)   │ │   (3)   │     │      │
│  │  │         │ │         │ │         │     │      │
│  │  │ 5-agent │ │capability│ │relation-│     │      │
│  │  │ memory  │ │discovery│ │  ship   │     │      │
│  │  │ system  │ │registry │ │  intel  │     │      │
│  │  └─────────┘ └─────────┘ └─────────┘     │      │
│  │                                           │      │
│  │           Agent Creation (0)              │      │
│  │           foundation layer                │      │
│  └───────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────┘
```

**Memory (Spec 1)** is the core — it's what makes your agent persistent. **Breadcrumbs (Spec 2)** and **CRM (Spec 3)** are independent systems that connect through the **Bridge (Spec 4)** without creating tight dependencies. The **Auditor (Spec 5)** verifies everything is wired correctly, and the **Lab Manager (Spec 6)** watches operational health over time.

---

## Getting Started

You don't need all seven specs on day one. Here's the recommended path:

### 1. Foundation
**Start with Spec 0 (Agent Creation)** — set up proper agent lifecycle management. This is the scaffolding everything else attaches to.

### 2. Add Persistence
**Add Spec 1 (Persistent Memory)** — give your agent memory that survives restarts. This is the single biggest upgrade you can make.

### 3. Add Self-Awareness
**Add Spec 2 (Breadcrumbs)** — let your agent discover its own capabilities. No more re-explaining what tools are available.

### 4. Connect the Systems
**Use Spec 4 (Integration Bridge)** — wire Memory and Breadcrumbs together without coupling them. The Bridge keeps things composable.

### 5. Optional: Relationship Tracking
**Add Spec 3 (CRM)** — if your agent interacts with multiple people, this is how it remembers who they are and what matters to them.

### 6. Verify Your Deployment
**Use Spec 5 (Deployment Auditor)** — automated checks that confirm everything is implemented correctly. Run it after setup, run it periodically.

### 7. Ongoing Health
**Use Spec 6 (Lab Manager)** — operational monitoring and tuning. This is what keeps a running system healthy over weeks and months.

---

## Version Strategy

Each spec has a **base version** and an optional **advanced version** (the `.1` releases):

| Base Version | What You Get | Advanced (.1) Version | What It Adds |
|:------------|:-------------|:---------------------|:-------------|
| v3 (Spec 0) | Solid agent lifecycle | — | — |
| v4 (Spec 1) | Full memory system | v4.1 | File locking, atomic writes, crash recovery |
| v3 (Spec 2) | Capability registry | v3.1 | Auto-discovery, stale breadcrumb cleanup |
| v2 (Spec 3) | Relationship tracking | v2.1 | Entity disambiguation, relationship graphs |
| v2 (Spec 4) | System integration | v2.1 | Event-driven sync, conflict resolution |
| v1 (Spec 5) | Deployment verification | — | — |
| v1 (Spec 6) | Health monitoring | v1.1 | Cron sentinels, automated tuning |

**Our recommendation:** Start with the base versions. They're stable, complete, and solve the core problems. Upgrade to `.1` versions when you hit the advanced use cases — you'll know when you need them.

---

## Who This Is For

**Primary audience:** [OpenClaw](https://github.com/nickhobbs94/openclaw) users who want agents that actually remember, learn, and operate reliably over time.

**But also:** Anyone building persistent agent systems. The patterns in these specs are framework-agnostic. We've seen similar problems (and similar solutions) across:

- **LangChain / LangGraph** — memory and tool management
- **AutoGPT / AgentGPT** — session continuity and self-improvement
- **CrewAI** — multi-agent coordination and knowledge sharing
- **Custom agent frameworks** — if you're rolling your own, these specs save you from rediscovering every pitfall

The spec documents describe *what* to build and *why*, with enough implementation detail to adapt to your stack.

---

## Contributing

This is a community project. We solved these problems for ourselves and figured others are hitting the same walls.

**Ways to contribute:**
- 🐛 **Found a gap?** Open an issue describing what's missing or what broke
- 💡 **Have an improvement?** PRs welcome — especially battle-tested patterns from your own deployments
- 📖 **Better docs?** Clarifications, examples, and diagrams always appreciated
- 🔧 **Reference implementations?** If you've built one of these specs in a specific framework, we'd love to link it

Please keep contributions focused on the specs themselves — architectural patterns and operational wisdom, not framework-specific code.

---

## License

[MIT](LICENSE) — use it, fork it, build on it. If these specs save you the weeks of debugging they saved us, that's the whole point.

---

<p align="center">
  <i>Built by people who got tired of their agents waking up with amnesia every morning.</i>
</p>
