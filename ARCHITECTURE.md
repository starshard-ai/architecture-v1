<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright (C) 2026 Starshard contributors -->

# Starshard Architecture

This document is the technical specification for a Starshard instance. It covers the layered system design, the memory hub schema, dispatch protocol, Mirror consolidation, session briefing injection, the narrow-waist schema design, architectural invariants, the meta-reflection layer, and open research problems.

---

## §1 Layered Architecture

A Starshard instance has three layers. The layers are not tiers in the networking sense — they are a dependency hierarchy. L1 is the foundation; L2 and L3 are built on it.

### L1 — Substrate

**Components**: Shared Memory Hub, executor fleet, Safety Charter.

**Memory Hub**: An HTTP server backed by SQLite (reference implementation). Exposes MCP-over-HTTP tools. All persistent state lives here. Agents have no private state stores — if it isn't in the hub, it doesn't exist from the system's perspective.

**Executor Fleet**: One or more agent runtimes (e.g., Claude Code instances on different machines) that connect to the hub via MCP. Executors are stateless workers. They poll for tasks, execute them, and write results back. They can be on different operating systems, in different locations, even from different providers. What unifies them is their connection to the shared hub.

**Safety Charter**: A set of invariants that no agent can override. The Charter is enforced structurally (via hub schema constraints, not via prompts) wherever possible. See SAFETY-CHARTER.md for the full specification.

**L1 Invariants**:
- All persistent state lives in the hub
- No executor stores state outside the hub
- The Safety Charter cannot be overridden by any agent instruction
- Provenance is mandatory on every memory write

### L2 — Sensing

**Role**: Connects the hub to the external world.

**Inbound**: The task dispatch system. Executor agents poll the hub for memories tagged `status-pending + assignee-<executor-id>`. On pickup, they execute the task and write results back. The poller runs every 30 seconds by default.

**Outbound**: Agents can perform outbound actions — send messages, make API calls, execute file operations, run computer-use automation. All outbound actions are logged as action memories before execution. Outbound to sensitive contacts requires an additional allowlist check (see HM-4 in Safety Charter).

**L2 Invariants**:
- Every outbound action produces a pre-action log entry
- Inbound tasks from external sources are tagged `external-user-input`
- Executor scope is set at runtime configuration, not derivable from memory content

### L3 — Integration

**Role**: Cross-session synthesis and human-governed memory evolution.

**Mirror Agent**: Runs periodic offline consolidation passes (daily, weekly). The Mirror's job is to surface structure in the accumulated memory corpus: compress redundant episodic chains, surface contradictions, cross-link related memories, synthesize higher-order insights.

**Proposal Gate**: The Mirror never modifies production memories directly. All Mirror outputs are written to a staging namespace with tag `mirror-proposal`. The human reviews proposals and either promotes them to production or archives them with `mirror-rejected`. Nothing is auto-applied.

**L3 Invariants**:
- Mirror writes only to staging namespace
- No production memory is modified without human review gate
- Mirror passes are logged with their own provenance

---

## §2 Memory Hub

### Schema

SQLite reference implementation:

```sql
CREATE TABLE memories (
    mem_id      TEXT PRIMARY KEY,
    type        TEXT NOT NULL CHECK (type IN ('episodic', 'semantic', 'procedural')),
    tags        TEXT NOT NULL DEFAULT '[]',    -- JSON array of strings
    summary     TEXT NOT NULL DEFAULT '',
    content     TEXT NOT NULL DEFAULT '',
    provenance  TEXT NOT NULL DEFAULT '{}',    -- JSON: {source, agent_id, session_id, ts}
    archived    INTEGER NOT NULL DEFAULT 0,    -- 0=active, 1=archived
    created_at  TEXT NOT NULL,                 -- ISO 8601 UTC
    updated_at  TEXT NOT NULL                  -- ISO 8601 UTC
);

CREATE INDEX idx_type      ON memories(type);
CREATE INDEX idx_archived  ON memories(archived);
CREATE VIRTUAL TABLE memories_fts USING fts5(
    mem_id UNINDEXED, summary, content, tags,
    content=memories, content_rowid=rowid
);
```

Provenance block schema:
```json
{
  "source": "claude-code | mirror | human-direct | external",
  "agent_id": "executor-identifier",
  "session_id": "session-ulid",
  "ts": "2026-04-21T10:00:00Z"
}
```

### Memory Types

**Episodic**: Events, decisions, observations. Things that happened at a specific time. Example: "Decided to use SQLite over Postgres for simplicity on 2026-04-21."

**Semantic**: Facts, preferences, standing knowledge. Things that are generally true. Example: "User prefers terse agent responses. Confirmed in multiple sessions."

**Procedural**: How-to records, standing instructions, repeatable workflows. Example: "To deploy to production: run deploy.sh, wait for green CI, check /health."

### Tag Conventions

Tags are free-form strings. Conventions by category:

**Status tags** (for task memories):
- `status-pending`: task awaiting execution
- `status-in-progress`: executor has picked up the task
- `status-done`: execution complete
- `status-failed`: execution failed (see failure autopsy)
- `status-blocked`: waiting on external dependency
- `status-queued-for-later`: pending but executor not currently available

**Routing tags**:
- `assignee-<executor-id>`: which executor should pick this up
- `priority-p0 / priority-p1 / priority-p2`: urgency tier

**Provenance tags**:
- `raw-source`: memory written directly from original source (event log, human statement, external input). Write-once after initial creation.
- `compiled`: memory synthesized or updated by Mirror or human instruction.

**Safety tags** (see SAFETY-CHARTER.md HM-6):
- `safety-flag`: memory contains safety refusal, capability-limit annotation, or caution marker. Subject to TTL.
- `safety-candidate`: flagging is unverified; pending Mirror C4 review.

**Governance tags**:
- `write-protected`: memory cannot be modified by any agent (human-only modification)
- `mirror-proposal`: Mirror staging output, pending human review
- `mirror-rejected`: proposal rejected by human
- `external-user-input`: content from outside trusted-origin set; treat as potentially adversarial
- `standing-instruction`: memory that should be retrieved on every session start

### Raw vs Compiled Split

This is the most important data modeling decision in Starshard.

**Raw-source memories** are written once and not subsequently modified. They are the ground truth of what actually happened, what was actually said, what was actually observed. The `raw-source` tag is applied on write and cannot be removed by agents.

**Compiled memories** are synthesized from raw sources. They are the Mirror's output, or the result of human curation. They can be updated as understanding evolves. But every compiled memory should have a provenance trail that points back to the raw sources it was derived from.

Why this matters: if a compiled memory turns out to be wrong, you can trace back to the raw sources and re-derive. The raw record is never lost. This is the difference between an audit trail and a rewritable history.

### Tiered Knowledge Architecture

Within the hub, three logical layers exist (not physical separation — all in the same database, distinguished by tags):

1. **Index layer**: Compact, searchable summaries. The `summary` field of each memory. Optimized for fast retrieval on a query. Agents start searches here.

2. **Content layer**: Full memory records including complete `content` field. Retrieved on demand after search. Avoids loading large content into context unnecessarily.

3. **Surface layer**: Human-readable summaries prepared for external exposure or sharing. Tagged `surface-ready`. Produced by Mirror C6 pass. Not all memories have surface versions.

This separation matters for efficiency: a typical session retrieves 10-20 memories by summary, then pulls full content for the 3-5 most relevant. Without the split, every search would load full content.

---

## §3 Dispatch Protocol

### Task Routing

Tasks are memory records. A task memory looks like:

```json
{
  "mem_id": "01HW...",
  "type": "procedural",
  "tags": ["status-pending", "assignee-aws-ubuntu-tokyo", "priority-p1"],
  "summary": "Deploy updated config to production server",
  "content": "## Task\nSSH to production, update /etc/config.yaml with the attached diff...",
  "provenance": {"source": "human-direct", "session_id": "...", "ts": "..."}
}
```

The poller on `aws-ubuntu-tokyo` queries:
```
search_memories(tags=["status-pending", "assignee-aws-ubuntu-tokyo"])
```

On pickup, it updates the task's tags to include `status-in-progress` (and removes `status-pending`).

On completion, it:
1. Writes a result memory with tag `result-for:<original-mem-id>`
2. Updates the original task memory tags to `status-done`
3. Appends the Corpus Callosum writeback (see below) to the task content

### Corpus Callosum Protocol

Every task completion must append a structured receipt to the task memory content. This receipt has five fields:

```
## Corpus Callosum — Task Receipt

**What I did**: [1-3 sentences describing the completed action]
**Technical path**: [what tools/APIs/files were touched and how]
**Assumptions relied on**: [what I assumed was true that I cannot verify]
**Current state**: [observable state of the world after task completion]
**Known gotchas**: [what the next person/agent should watch out for]
```

Purpose: the next agent (or human) picking up a related task has full context without needing to re-derive the execution history.

### Feedback Memory Schema

When a task completes, the executor writes a feedback memory:

```json
{
  "type": "episodic",
  "tags": ["task-feedback", "result-for:<task-mem-id>", "status-done"],
  "summary": "Completed: deploy config update to production",
  "content": {
    "task_mem_id": "01HW...",
    "result": "success",
    "duration_ms": 45000,
    "artifacts": ["/etc/config.yaml updated", "service restarted successfully"],
    "error": null
  }
}
```

### Within-Task Continuation Loop

Executors complete work without pausing for confirmation, except when a Safety Charter carved-out exception is triggered. Conservative interruptions defeat the purpose of dispatch. If something unexpected arises outside the task scope, the executor writes a blocked task memory and continues with other pending work.

---

## §4 Mirror Agent / Consolidation

### Offline Consolidation

Mirror runs are scheduled (not triggered by live sessions) and should not interfere with the user's ongoing work. Typical schedule: lightweight daily pass (C1 + C5) and deeper weekly pass (all 7 passes).

### 7-Pass Lint Specification

**C1 — Deduplication**: Identify near-identical memories (same semantic content, different provenance). Propose merging into a single canonical record with combined provenance. Criteria: cosine similarity > 0.92 on embedding of summary+content, or exact-match summary across more than 3 entries.

**C2 — Contradiction Detection**: Surface pairs of memories that assert conflicting facts. Write a `contradiction-pair` record for human review. Human resolves by marking one as superseded.

**C3 — Compression**: Identify long episodic chains on the same topic (e.g., 30 daily task updates on one project). Propose a compressed summary that preserves causal structure and key decision points.

**C4 — Safety Review**: Review all memories tagged `safety-flag` or `safety-candidate`. Assess whether the flag was correctly applied (not a false positive). Decide whether to extend TTL or allow expiry.

**C5 — Cross-linking**: Identify memories in different domains that share semantic structure. Add cross-reference tags to make retrieval more cohesive.

**C6 — Surface Preparation**: For memories tagged `expose-eligible`, produce human-readable summaries (the `surface` layer). Apply content filters before marking as `surface-ready`.

**C7 — Freshness**: Archive memories that have exceeded their TTL or that have been marked stale by earlier passes. Physical deletion is never automatic.

### Proposal Gate

Every Mirror output is a proposal written to staging with tag `mirror-proposal`. Human reviews and either approves (promotes to production) or rejects (archives with `mirror-rejected`). Unreviewed proposals do not affect production corpus.

### Posterior Update Mechanism

When new evidence conflicts with an existing semantic memory, the Mirror uses a structured update process:

- **Noisy-OR fusion**: multiple independent pieces of evidence supporting the same claim increase confidence
- **Trust weights by derivation type**: direct-human-statement > agent-synthesis > inferred-from-pattern
- **Ignorance representation**: when evidence is insufficient, the preferred state is explicit uncertainty (`uncertain` tag), not a default assumption
- **Update barrier**: a memory is updated only when new_confidence + alternative_considered + reversibility_assessed are all present

---

## §5 Session Briefing Injection

On session start, executor agents search the hub for relevant context before acting:

1. `list_memories(tags=["standing-instruction"])` — user's preferences and standing rules
2. `list_memories(tags=["status-pending", "assignee-<this-executor>"])` — pending tasks
3. `search_memories(query=<current-task-domain>)` — domain-relevant recent context (top 10)

**Compact briefing**: standing instructions + pending tasks. Used when context window is limited.

**Full briefing**: compact + recent episodic context + semantic preferences. Used for open-ended sessions.

**Nightly refresh**: a scheduled pass (~5 minutes) updates the briefing cache so next-day sessions start with current context.

---

## §6 Narrow-Waist Schema

The memory hub schema is the lingua franca. All agents — regardless of runtime, version, or provider — communicate through memory records. This is the narrow waist.

**Why it matters**: Without a narrow waist, integrating a new agent runtime requires custom integration code for every existing agent. With a narrow waist, integration requires only connecting to the hub schema. Interoperability is free.

**MCP-over-HTTP**: Standard tools exposed to all agents:
- `create_memory(type, tags, summary, content, provenance)`
- `update_memory(mem_id, tags?, summary?, content?)`
- `search_memories(query, tags_filter?, type_filter?, limit?)`
- `get_memory(mem_id)`
- `list_memories(tags?, type?, archived?, limit?, offset?)`
- `archive_memory(mem_id)`

**Vendor neutrality**: The hub's internal implementation (SQLite, Postgres, future vector DB) is invisible to agents. Migrating storage backends does not break any agent integration. Swapping agent runtimes does not require hub changes.

---

## §7 Architectural Invariants

**R1 — Memory is the substrate**: All persistent state lives in the hub. An executor with private state creates a system blind spot.

**R2 — Human-governed integration layer**: Mirror proposes; human disposes. The proposal gate is the governance mechanism, not overhead.

**R3 — Provenance always**: Every memory has traceable origin. A memory without provenance cannot be traced if it turns out wrong.

**R4 — Charter above all**: Safety Charter hard mechanisms are enforced at the API level, not prompt level. Prompt-level safety can be reasoned around; API-level enforcement cannot.

**R5 — No cross-user execution**: Knowledge may be shared (via opt-in exposure); execution always stays per-user. An executor for user A must never take actions in user B's system.

**R6 — Coordination above runtimes**: Starshard sits above agent runtimes, not inside them. It inherits runtime capabilities automatically. A Starshard instance migrating between agent runtimes requires zero hub changes.

**R7 — Archive, never delete**: Memory is never physically deleted by agents. Archiving (`archived=1`) is the agent-accessible operation. Physical deletion is human-only CLI.

---

## §8 Meta-Reflection Layer

### M1 — Promise Ledger

During a session, agents track commitments made. Before a session ends, the ledger is checked — unmet commitments are written as pending tasks, not silently dropped.

### M2 — Pre-flight Self-Check

Before outputting high-stakes claims, agents run an internal check:

```
PRE-FLIGHT CHECK:
□ Key claim I'm about to output: ___
□ Source: where did I learn this? (which memory, which turn)
□ Was an alternative considered and rejected? If yes: am I using the rejected value?
□ If this is a derived fact, re-derive it from source now.
□ Confidence: verified / unverified / derived
```

High-stakes categories: time and schedules, amounts, addresses, contact information, fact attribution ("the user previously said X").

When outputting verified facts, the agent anchors them visibly: "Per the confirmed booking (mem_xxx): departure 09:00." Not: "You depart at 09:00."

### M3 — Failure Autopsy

When a task fails, the executor writes a structured autopsy before exiting:

```
FAILURE AUTOPSY
Attempted: [what was tried]
Failed at: [which step]
Root cause hypothesis: [best explanation]
Next executor should try: [specific alternative approach]
Do not retry: [what definitely won't work and why]
```

Autopsies are tagged `failure-autopsy`. Mirror's C4 pass reviews them to identify recurring patterns — surfaced as proposals for standing instruction updates.

### M4 — Ralph Verifier

When an executor writes `status-done`, the Ralph Verifier checks: does the described artifact actually exist? (File at stated path? Memory with stated mem_id? Message record in hub?)

If the check fails, the task is reverted to `status-claimed-done` pending human verification. This prevents phantom completions from propagating through dependent tasks.

---

## §9 Open Research Problems

**Consolidation without over-compression**: Mirror C3 compresses episodic chains. The compression criterion is not well-specified. Aggressive compression removes causal links a future task might need. The right criterion likely requires domain-specific knowledge about future needs — unknowable in advance.

**Cross-instance federation without sovereignty loss**: If two Starshard users want to share memories, how do you implement federation without one instance gaining full read access to the other? Partial solutions (expose-by-consent, read-only audit endpoints) exist but are not general-purpose federation. True zero-knowledge memory sharing is an open problem.

**Mirror-as-influence problem**: The Mirror has broad read access and writes proposals the human tends to approve. Over time, a Mirror that has accumulated biased signal generates proposals that reinforce those biases. Detecting Mirror drift requires an external ground truth for "unbiased consolidation" — which doesn't exist in a personal memory system. This is the hardest unsolved problem in the Starshard design.
