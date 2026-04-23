<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright (C) 2026 Starshard contributors -->

# Starshard

**A shard that holds the whole.**

Starshard is a substrate for multi-agent personal AI systems where memory is the primary layer and agents are peripherals. Every user runs their own complete instance. Data is owned by the user. Architecture prioritizes session-spanning coherence over session-level stickiness.

---

## The Problem

Current AI tools have a memory problem. Each session starts fresh. The assistant that helped you draft a document last Tuesday knows nothing about the decision you made last Thursday. Every conversation reinvents context. State lives in the human's head, not the system's.

This is worse than 1995 desktop email, which at minimum persisted a searchable archive across sessions.

The culprit is an architectural assumption: that the *agent* is the system, and memory is an optional add-on. This assumption produces tools that are individually capable but collectively amnesiac. Add more agents and the problem compounds — each one brilliant in isolation, none of them aware of what the others have learned.

## The Inversion

Starshard inverts the stack.

**Memory is the substrate.** Agents are peripherals. Every insight, decision, task, and preference written by any agent goes into a shared, queryable hub. Every future agent session reads from that hub before acting. The hub is the persistent identity of the system; agents are stateless workers that serve it.

This produces a qualitatively different system. A user who sets up Starshard accumulates a growing record of their preferences, prior decisions, ongoing tasks, and hard-won knowledge. Future sessions inherit all of it automatically. The system gets *better* over time, without any retraining, fine-tuning, or provider-side personalization.

## Why This Exists

Three observations motivate Starshard:

**Observation 1 — Session-stickiness is not coherence.** Commercial AI products optimize for "good session" metrics: user satisfaction per conversation. This is not the same as "good system over time." A system can produce excellent individual sessions while accumulating zero compounding value across them. Starshard optimizes for compound coherence — each session adds to a structure that makes future sessions better.

**Observation 2 — Human memory is the bottleneck.** In current human-AI workflows, the human carries all cross-session state: they remember what they asked for, what worked, what the agent got wrong. This is a hidden tax on every interaction. The human becomes a context-relay. Starshard moves that burden to the substrate, freeing the human to think rather than recap.

**Observation 3 — Data sovereignty matters more as the system deepens.** A memory hub that accumulates years of a user's preferences, relationships, and decisions is a sensitive asset. Starshard's design assumes the user must own and control this data. The hub runs on the user's own infrastructure (or a VPS they control), behind their own authentication layer, with no third-party cloud storage. This is not a privacy feature — it is an architectural commitment.

## How It's Different

### vs. native AI memory features (ChatGPT memory, Claude projects, Gemini context)

Provider-managed memory is stored by the provider, readable by the provider, subject to the provider's retention policies, and lost if you switch providers. It optimizes for within-provider continuity. Starshard optimizes for provider-independence: the memory hub is yours, runs anywhere, and works with any agent runtime that can make HTTP calls.

### vs. custom context in system prompts

Manual context injection (pasting prior notes into system prompts) works but is labor-intensive, error-prone, and does not scale. Starshard automates context injection via session briefing — the agent searches the hub before acting. The human's job is not to maintain a context document; it is to live in the system.

### vs. personal knowledge management tools (Obsidian, Notion, Roam)

PKM tools store human-authored knowledge. They do not accumulate AI-authored context, task results, or structured decisions. They lack task dispatch, multi-agent coordination, or the proposal-gate consolidation pattern. Starshard is complementary to PKM tools — not a replacement.

### vs. multi-agent frameworks (AutoGen, CrewAI, LangGraph)

Agent frameworks solve coordination. Starshard solves persistence. A multi-agent framework with Starshard as its memory layer gets both. A multi-agent framework without persistent memory restarts from zero every run.

## Architecture in Brief

A Starshard instance has three layers:

**L1 — Substrate**: The Shared Memory Hub is an HTTP-accessible database of structured memory records. A fleet of executor agents (AI runtimes like Claude Code) connects to it. A Safety Charter defines invariants that no agent can violate.

**L2 — Sensing**: Inbound and outbound task automation connects the hub to the external world. Agents poll for pending tasks, execute them, and write results back as memories. This layer handles things like: scheduled information gathering, outbound notifications, file operations, and computer-use automation.

**L3 — Integration**: A Mirror agent (consolidation component) runs periodic offline passes over accumulated memories, compresses redundant information, resolves contradictions, and synthesizes higher-order insights. The Mirror proposes; the human disposes. Nothing is auto-modified without a human review gate.

Full architectural detail: [ARCHITECTURE.md](ARCHITECTURE.md)

## Quick Start

See [QUICKSTART.md](QUICKSTART.md) for the full self-hosting guide.

The short version:
1. Run the hub server (FastAPI + SQLite, ~50 MB total footprint)
2. Expose it behind Cloudflare Tunnel + Cloudflare Access (free tier)
3. Add it to `~/.claude/mcp.json` as an MCP server
4. Start a Claude Code session — it can now read and write persistent memories

Phase 0 (memory hub only) takes 30-45 minutes to set up. Phase 1 (task dispatch) adds a poller daemon and takes another hour.

## Project Status

Starshard is an architecture specification and reference implementation, released for community adoption and feedback.

**What is complete:**
- Memory hub schema and API specification (ARCHITECTURE.md §2)
- Dispatch protocol specification (ARCHITECTURE.md §3)
- Mirror consolidation specification (ARCHITECTURE.md §4)
- Safety Charter with 6 hard mechanisms (SAFETY-CHARTER.md)
- Self-hosting guide covering Phase 0 through Phase 2 (QUICKSTART.md)
- Reference hub implementation (Python + FastAPI + SQLite)
- Poller daemon reference implementation

**What is in progress:**
- Full test suite for the reference implementation
- CLI tooling (`starshard-hub` management commands)
- Mirror consolidation passes C1-C3 reference implementation

**What is planned:**
- Vector search integration for semantic retrieval (Phase 1 enhancement)
- Web dashboard for memory review and Mirror proposal queue
- Federation protocol specification (post-v0)

## Licensing

Starshard uses a hybrid license stack:

| Component | License |
|-----------|---------|
| Architecture docs (ARCHITECTURE.md, QUICKSTART.md, README.md) | Apache 2.0 |
| Safety Charter (SAFETY-CHARTER.md) | CC0 1.0 — maximum diffusion, no restrictions |
| Philosophy (PHILOSOPHY.md) | CC BY-SA 4.0 |
| Hub server code (future) | AGPL v3 |
| CLI utilities (future) | Apache 2.0 |
| Schema definitions (future) | CC0 1.0 |

CC0 for the Safety Charter is intentional: threat models and hard safety mechanisms should be freely adoptable by any personal AI system without attribution requirements.

## Related Work

Starshard was preceded by a LessWrong post describing the memory consolidation thesis:
[Starshard: Sleep-Inspired Memory Consolidation for a Multi-Agent Personal AI](https://www.lesswrong.com/posts/wg56edFhuPCsZrncQ/starshard-sleep-inspired-memory-consolidation-for-a-multi)

That post covers the theoretical foundations in more depth: why sleep-inspired consolidation maps onto multi-session AI memory, what the "forgetting problem" looks like at scale, and open research questions. The architecture documents in this repository are the operational follow-on.

## Contact

Questions, feedback, and collaboration inquiries: macshen93@gmail.com

## Contributing

- **Issues**: Welcome. File bugs, spec clarifications, and design questions as GitHub Issues.
- **Pull requests**: Accepted for documentation, spec improvements, and bug fixes. Open an issue first for significant changes.
- **Philosophical discussions**: Belong in LessWrong comments, not GitHub Issues. Architecture debates are fine here; foundational disagreements about whether personal AI memory is good are not.

---

*"A shard that holds the whole" means each user's instance is complete — not a fragment awaiting connection to a central server. The holographic framing is intentional: the full pattern exists in each piece. Federation is overlap, not dependency.*
