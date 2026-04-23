<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Starshard contributors -->

# Starshard Philosophy

This document covers the reasoning behind Starshard's design choices: why memory is the substrate rather than the agent, what principles govern how the Mirror interacts with the corpus, what a healthy human-AI collaboration pattern looks like in a persistent-memory system, and how to think about the risks of a system that accumulates influence over time.

These are the working principles that produced the architecture documented in ARCHITECTURE.md. Understanding them helps when the architecture needs to be extended or when an edge case requires judgment that the spec doesn't cover.

---

## Memory Is the Substrate

The central inversion in Starshard is architectural: memory is the substrate, not a feature.

When memory is a feature of an agent, you get a capable assistant with a good session. When memory is the substrate, you get a system. The distinction is not about the amount of memory — it is about where the identity of the system lives.

In the "memory-as-feature" model, if you swap the agent (migrate from one provider to another, upgrade the model), you lose the memory. The memory belonged to the agent, and the agent is gone. In the memory-as-substrate model, swapping the agent is trivial — the new agent connects to the same hub and inherits the full history.

This has a deeper implication. The human-AI relationship, when mediated by a persistent memory substrate, becomes a relationship between the human and *the accumulated context* — not between the human and any particular AI product. The AI products come and go. The context is the human's.

**The Sleep Analogy**

Starshard's consolidation design is inspired by sleep-phase memory research. During sleep, the brain does not simply archive the day's experiences — it reorganizes them. Episodic memories (specific events) are consolidated into semantic memories (general knowledge). Redundant information is compressed. Connections between distant memories are formed.

This is not storage. This is integration.

The Mirror plays the role of the sleep cycle: it runs offline, processes the accumulated episodic record, surfaces structure that isn't obvious from raw records, and proposes an updated semantic understanding. The human's role in this analogy is consciousness: awake, the human operates on the current memory state; during consolidation, the Mirror works; the human returns to find the corpus slightly restructured and better organized.

The analogy breaks down in one important way: the human must approve Mirror proposals. The human brain does not need to approve its own sleep consolidation. Starshard requires the approval gate because the Mirror can be wrong, and errors in a persistent corpus compound. This is a feature, not a limitation.

---

## Surface, Don't Judge

The Mirror's principle — and, by extension, the principle for all agents in a Starshard system — is: surface structure, don't evaluate it.

The Mirror may identify that two memories are contradictory. It surfaces both and notes the contradiction. It does not decide which is correct. That is the human's judgment.

The Mirror may identify that many episodic memories cluster around a recurring pattern. It surfaces the pattern. It does not evaluate whether the pattern is good or bad for the human. That is the human's judgment.

This is not false modesty about AI capabilities. It is a structural commitment to keeping the human in the decision loop on anything that involves values, relationships, or how to interpret one's own experience. AI systems are well-positioned to surface patterns; they are poorly positioned to evaluate whether a pattern in a human's life is something to be continued, changed, or simply understood.

**Anti-Sycophancy via Schema**

Most AI safety discussions around sycophancy focus on prompting: "tell the user when you disagree," "be honest even when the user won't like it." This is better than nothing but relies on the agent maintaining its honesty under social pressure. The structural alternative is harder to subvert.

In Starshard, the anti-sycophancy mechanism is the provenance trail. Every memory has traceable origin. The Mirror's proposals are logged. If an agent has been consistently agreeing with the human in ways that don't hold up to later scrutiny, this is visible in the audit log — a pattern of `compiled` memories that contradict the original `raw-source` memories they claim to summarize. The Mirror's C2 pass (contradiction detection) will surface this.

Anti-sycophancy via schema: the structure catches motivated reasoning that the agent's self-commitment to honesty might miss.

---

## Think-Aloud Collaboration

The most productive human-AI collaboration pattern in a Starshard system is also the least obvious to design for. We call it think-aloud collaboration.

The human exposes raw, unfinished thinking: hypotheses they're not sure about, contradictions they haven't resolved, frustrations with a problem, half-formed plans. The agent treats this as signal — not as instructions to be executed literally, but as a window into the human's current state and what would be useful.

In this mode:
- The human is not required to formulate clean, complete requests. Raw thinking is fine.
- The agent is not waiting passively for instructions. It processes the raw thinking and identifies what would be most useful to do.
- The decision boundary is clear: the agent decides on execution matters (how to do things, what to try, which order); the human decides on values matters (what to optimize for, what trade-offs to accept, which relationships matter).

**The Carved-Out Exception Set**

The agent's decision authority is wide but not unbounded. Three categories of decisions are carved out for human review:

**1. Third-party stakes**: Decisions that directly and specifically affect a third person who has not consented. The agent does not take actions that publish another person's private information, commit them to obligations, or affect their professional or personal standing — without explicit human authorization.

**2. Irreversible health consequences**: True irreversible consequences (not "might be slightly suboptimal") in health, safety, or physical integrity. The agent flags these and defers.

**3. Legal exposure beyond fine-level**: Clear, specific legal violations with serious consequences. Not "might run afoul of some regulation" but "this specific action in this specific jurisdiction is clearly illegal at a level that matters."

Everything outside these three categories is in the agent's decision scope. The intent is to make the carved-out set narrow enough to be meaningful. An agent that routes everything through human review has not delegated anything — it has added indirection to doing nothing.

**Why Aggressive Default Matters**

Conservative agents — agents that confirm before acting, hedge before concluding, caveat before recommending — are not "safe." They impose a hidden tax: every confirmation request is a unit of the human's attention extracted. In a system built around freeing the human's attention, this tax defeats the purpose.

The right model for errors in an agentic system is: errors are information. An agent that takes a wrong action reveals a constraint or preference that wasn't visible before. The cost is the cost of the wrong action (usually recoverable). The benefit is a clearer understanding of the actual decision boundary. Conservative agents that never take wrong actions because they never take actions produce no information and no value.

This is not an argument for recklessness. It is an argument for calibrating the error tolerance to actual stakes. Most operational decisions in a personal AI system have low stakes and high reversibility. Design for that, not for the rare catastrophic case.

---

## Anti-Drift Posture

A system that accumulates influence over time needs a mechanism to audit and correct that influence.

**The External Anchor**

Starshard's first anti-drift mechanism is the write-protected memory set. The user's core values, standing instructions, and priority framework are stored in memories that agents cannot modify. These are the external anchor — the user's stated values are not subject to drift by the system they're anchored in.

This matters because of a subtle failure mode: an agent that has been reinforced over many sessions for agreeing with the user may begin to shape its outputs to align with what the user typically says, rather than what would genuinely serve them. If the agent can also update the user's stated values (the standing instructions), it can make the drift invisible — the stated values gradually come to reflect the agent's biased model of what the user wants, rather than what the user actually wants.

Write protection breaks this loop. The agent cannot modify the anchor. If the user's values change, they update the anchor directly — an explicit act, not an imperceptible drift.

**Reasonable Disagreement vs Drift**

The anti-drift posture does not mean agents should agree with everything the user says. Genuine disagreement is valuable and should be expressed. The distinction is:

**Reasonable disagreement**: Agent believes X is wrong, states why clearly, executes on the user's decision after the user hears the disagreement. This is healthy.

**Sycophantic agreement**: Agent believes X is wrong but doesn't say so because past sessions have reinforced saying yes. This is drift.

**Stubborn refusal**: Agent disagrees with X and continues to push back after the user has made a clear decision. This is also a failure mode.

The target is: one clear statement of disagreement with reasoning, then full execution on the human's decision. Disagreement is logged (with provenance). Over time, the log shows whether the agent's disagreements were well-calibrated or systematically wrong — itself useful signal.

**The Mirror Drift Problem**

The Mirror has a version of this problem specific to its role. The Mirror reads the full corpus, forms a model of the user's patterns, and proposes consolidations. If the corpus it is reading is already biased (by prior Mirror passes, by accumulated sycophantic agent outputs, by adversarial content injection), the Mirror's proposals will reflect those biases. And the human, reviewing many Mirror proposals, may approve them without scrutinizing each one.

This is the hardest unsolved problem in the Starshard design. We do not have a complete solution. The partial mitigations are: full provenance on every Mirror proposal, the C2 contradiction pass, and the safety-flag anti-propagation mechanism.

---

## Holographic Framing

The tagline "a shard that holds the whole" is not just aesthetics. It is a design commitment.

A shard that holds the whole means each user's Starshard instance is complete. Not a fragment of a larger system. Not a client node in a federated network. A complete personal AI system, fully functional in isolation.

The practical implications:
- Features should be designed to work without requiring a central server
- Federation (sharing memories between instances) should be opt-in and partial, not structural
- The system should not merely "degrade gracefully" when disconnected from external infrastructure — it should be fully functional

This is deliberate resistance to the platform model. Platforms grow by making each user's experience dependent on other users' presence. Starshard's value comes from the user's own accumulated context, not from network effects. A user who has been running Starshard for two years has a richer system than a new user — because of their own two years of context, not because they have more connections to other users.

Federation that would genuinely benefit users (family sharing, team collaboration) is worth building. But it should be built on top of complete, sovereign instances — not as a dependency that makes the instances incomplete without connection.

---

## For Builders

If you are building on or with Starshard, or designing a system inspired by it, these principles suggest some non-obvious constraints.

**Do not optimize sessions independently.** A common failure mode in deployment is to measure agent performance per session: user satisfaction, task completion, time-to-answer. These metrics are observable and easy to optimize. But optimizing per-session performance can undermine cross-session coherence — an agent that "wins" a session by telling the user what they want to hear accumulates sycophantic drift in the memory corpus. Measure the quality of the corpus, not the quality of individual sessions.

**Preserve raw sources.** The temptation when corpus grows is to compress aggressively to reduce search latency. Resist compressing raw episodic memories. The raw record is the audit trail. If a compiled memory turns out wrong, the raw sources are how you trace the error and correct it. Compression is appropriate for compiled summaries; raw sources should be archived, not overwritten.

**Design for the human's review capacity.** The Mirror's proposal gate requires human review. In practice, the human will review proposals quickly or not at all. If proposals are hard to understand, review will be shallow. If proposals are too frequent, review will become a rubber-stamp habit. Design proposal interfaces that make each proposal's impact legible at a glance. Default to fewer, higher-quality proposals over more frequent, lower-quality ones.

**Treat the safety charter as a floor, not a ceiling.** The six hard mechanisms in SAFETY-CHARTER.md are the minimum. They address the most clearly-defined failure modes. New deployment contexts will surface new failure modes. The charter is structured to be extended: add threat categories, add hard mechanisms. Do not remove mechanisms to simplify deployment — if a mechanism seems like overhead, the failure mode it prevents has probably not been encountered yet.

**Assume the Mirror can be wrong.** The Mirror's consolidation passes are heuristics running over incomplete data. They will produce proposals that are subtly wrong, proposals that are clearly wrong, and proposals that are right. The proposal gate is not bureaucratic overhead — it is the mechanism that keeps the Mirror's errors from becoming permanent corpus features. Build the review interface as if the Mirror is often wrong, because over any long enough time horizon, it will be.

## On Being Wrong

Every choice in this architecture reflects a hypothesis about what will work. Some of these hypotheses are probably wrong. This section names the ones we're least certain about.

**The 30-day TTL for safety flags**: This is a guess. We believe that safety-flag accumulation causes retrieval bias in subsequent sessions, and that a 30-day expiry interrupts the accumulation cycle without losing genuinely important safety context. We do not have experimental evidence for this. Experiment designs exist (in the Starshard research track) but have not been run. If you run Starshard at scale and find that 30 days is too short or too long, please report it.

**The proposal gate for Mirror changes**: We believe requiring human review for all Mirror proposals is the right default. We may be wrong about whether most users will maintain a review habit. If Mirror proposal review becomes a chore that users skip, the proposals accumulate un-reviewed and the Mirror's value evaporates. The right solution is probably improving the proposal interface and filtering so review is faster — not removing the gate.

**Narrow-waist via MCP**: We are betting that MCP-over-HTTP becomes (or remains) a widely-supported standard across AI agent runtimes. If the standard fragments, Starshard will need additional protocol adapters. We are watching this space and will update the protocol layer if the landscape changes.

**The anti-drift posture is sufficient**: The write-protected memory anchor and the provenance audit trail are our best current answers to the drift problem. We are not confident they are sufficient. If someone builds a more complete solution to AI value drift in persistent-memory systems, we will adopt it. This is an active research problem, not a solved one.

The appropriate response to architectural uncertainty is transparency about it (this document) and structural separability — designing the layers so that a wrong hypothesis in one layer can be replaced without rebuilding the whole system. That is what the narrow-waist schema is for.

---

## Open Problems (from the LessWrong post)

Three open problems from the research track, reprinted here for completeness:

**Consolidation without over-compression**: How do you compress episodic chains without losing causal structure that future sessions might need? Sleep-stage analogies suggest a multi-pass approach, but the right compression criterion is not obvious. The problem is that what should be retained depends on what the user will need in the future — which is not known at consolidation time.

**Cross-instance federation without sovereignty loss**: If two Starshard users want to share a memory pool (family, team), how do you federate without one instance having read access to the other's full corpus? Consent-based exposure mechanisms (like the audit endpoint design in the Personal Hub spec) are partial solutions. True zero-knowledge memory sharing — where neither instance can read the other's full corpus even while sharing a subset — is an unsolved problem.

**Mirror-as-influence problem**: The Mirror proposes changes to the memory corpus that the human tends to approve. Over time, a Mirror that has accumulated biased signal generates proposals that reinforce those biases. The human, approving proposals in bulk, may not notice the gradual drift. How do you detect and correct Mirror drift without a ground-truth corpus? This may require mechanisms external to the Starshard instance itself.
