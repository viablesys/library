# Meta-Cognition — Thinking About Thinking in Agent Systems

How an agent system observes, records, and reasons about its own cognitive processes. Applied to memory systems (nmem), hook-based observation, and session continuity.

Written speculatively before nmem was built. Revised after ~10 sessions of building and using the system against its own data.

## Core Concepts

### Cognitive Loop

An agent operates in a loop: receive intent → reason → act → observe result → reason again. Each stage produces artifacts:

| Stage | Artifact | nmem Mapping |
|-------|----------|-------------|
| Receive intent | User prompt | `prompts` (source=user) |
| Reason | Thinking block | `prompts` (source=agent) |
| Act | Tool call | `observations` |
| Observe result | Tool response | `metadata.response` (failures only) |
| Reason again | Next thinking block | `prompts` (source=agent) |

The memory system captures *intent* (why), *action* (what), and *outcome* (whether it worked). The original design discarded tool responses entirely as ephemeral. Building nmem revealed this was too aggressive — failure responses carry strong signal. A build that fails is more informative than a build that succeeds. PostToolUseFailure responses are now stored in observation metadata.

Successful tool responses remain discarded — their content (file text, command output) is in the transcript and too large to store. The compromise: capture the *fact* of success or failure, not the full perception.

### Cognition as Recursive Viable Systems

Early framing used a linear progression (Level 0–4). Implementation revealed that cognition in agent systems is better understood as recursive viable systems (VSM), where each layer is itself a viable system with its own observation-reasoning-action loop.

**The original levels (still useful as capability descriptions):**

| Level | Capability | Example |
|-------|-----------|---------|
| 0 — Reactive | No memory | Claude Code without nmem |
| 1 — Observational | Record raw events | Session transcripts |
| 2 — Structured | Classify, link intent to action | nmem observations + prompt_id |
| 3 — Reflective | Cross-session pattern recognition | "These three sessions all touched auth" |
| 4 — Anticipatory | Predict needed context | Inject past decisions relevant to current work |

**What implementation taught:** These are not a ladder. nmem jumped to Level 2 immediately, is designing Level 4 (anticipatory context injection) before Level 3 (cross-session reflection) is mature, and skipped Level 1 entirely. The levels describe independent capabilities, not a sequence.

More importantly, the levels don't capture *nesting*. S1 (operations) is itself a viable system — it has its own S4 (session summarization) that compresses what happened *within* a session. The outer S4 synthesizes *across* sessions. S1's S4 must work before the outer S4 can build on it. This recursion is invisible in the flat level model.

**The recursive framing:**

```
S1 captures observations (Level 2)
  └─ S1's S4 compresses within a session (Level 2→3 transition)
       └─ summaries feed the outer S4
            └─ outer S4 synthesizes across sessions (Level 3→4)
                 └─ context injection shapes the next session's behavior
                      └─ behavior generates new observations (loop closes)
```

Each layer is a viable system that observes the layer below through its own lens (SQL views, not shared tables — see Inter-System Observation below).

### Self-Referential Observation

nmem observes its own design process. The capture tool records hook events from sessions where the capture tool itself was being built and refined. The prototype DB contains observations about creating the prototype DB.

This creates a productive recursion:
1. Build a capture tool → generates data about how the tool was built
2. Analyze that data → reveals patterns that inform the schema
3. Build a schema → tested against the data from step 1
4. Schema captures future sessions → including sessions where the schema is refined

The recursion grounds design decisions in real data rather than hypothetical requirements. Concrete example: the original volume estimate (5 MB/year) was off by two orders of magnitude. Live production data showed ~652 MB/year, driven by agent thinking blocks (84% of content volume). This was discovered by querying nmem's own database during a session that nmem was recording. The finding immediately changed the design: compression (session summarization) became a year-1 necessity, not a distant concern.

### Intent Hierarchy

Not all intents are equal. They form a hierarchy:

- **Strategic**: "Let's build nmem" (project-level, spans sessions)
- **Tactical**: "Add session signatures to ADR-003" (task-level, spans a work unit)
- **Operational**: "Read the file first" (step-level, spans one tool call)

User prompts tend toward tactical. Agent reasoning tends toward operational. The strategic level is implicit — derived from the accumulation of tactical decisions across sessions, not from any single prompt.

nmem's `prompts` table captures tactical and operational intent. The `prompt_id` foreign key linking observations to their causal prompt turned out to be the single most important schema decision — it's what makes intent-weighted context injection work. Strategic intent emerges from retrieval: aggregating tactical observations across sessions to answer "what have I been working on?"

### Forgetting as Compression

A memory system that stores everything has perfect recall but no understanding. Understanding requires forgetting — or more precisely, *compression*.

The compression pipeline in practice:

| Stage | Compression | Ratio |
|-------|------------|-------|
| Raw events → structured observations | Dedup, classification, truncation | ~1:1 (most events stored) |
| Observations → session summaries | LLM summarization at session end | ~200:1 (200 observations → 1 summary) |
| Session summaries → context injection | Scoring, filtering, limit caps | ~10:1 (10 sessions → top summaries) |

Each stage is lossy. The lossy steps are where judgment happens — what to keep, what to collapse, what to discard. This is meta-cognition: the system deciding what's worth remembering.

Key finding: narrative compression requires language generation. Structured templates from observations alone can't capture intent or causality. Even a small local LLM (granite-4-h-tiny) produces useful compression when the prompt is correctly framed — specifically, framing the consumer as the next AI session, not a human reader.

### Cross-Session Identity

Without persistent memory, each session is a new agent. It has access to code and docs but no experiential continuity — it doesn't know what it tried yesterday, what worked, what failed.

nmem's SessionStart context injection bridges this gap. The injected context is now summary-primary: session summaries (with `learned` and `next_steps` fields) are the main signal, supplemented by pinned observations, recent file edits, and git milestones. This is a form of identity continuity — not full consciousness, but enough to avoid repeating mistakes and build on prior work.

The quality of this continuity depends entirely on what gets injected. Too much context and the agent drowns in noise. Too little and it starts cold. The right content is: recent intent (what was I doing?), learned decisions (what did I conclude?), and next steps (what's still open?). Raw observation lists are noise — summaries are signal.

## Signal Multiplication

Observation signals and summary signals are not independent inputs. They multiply.

**Layer 1 — Micro (observations):** Per-tool-call facts. Tool composition ratios per prompt, file access patterns, failure sequences, timestamps. These detect work unit *boundaries* within a session.

**Layer 2 — Macro (summaries):** Per-session structured semantics. Intent, learned decisions, completed work, next steps, failure notes. These detect *patterns* across sessions.

**Neither layer is sufficient alone:**
- Micro signals detect boundaries but don't know meaning. A shift from reads to edits is a phase transition, but which past decisions are relevant?
- Macro signals carry meaning but are coarse. A session summary says "JWT middleware preferred over guard" but doesn't know when to surface it.

**Multiplied:** The micro layer's hot files (current work unit touches `src/auth.rs`) select against the macro layer's learned insights (previous session's `files_edited` overlap + `learned` mentions middleware). The result is targeted retrieval — not "here's everything from your last session" but "here's the specific decision relevant to what you're doing right now."

This is the difference between time-ordered context injection (what nmem does now) and relevance-ordered injection (what S4 enables). The multiplication is relational (SQL joins over existing tables), not semantic (no vector search needed).

## Inter-System Observation

Higher cognitive layers need to observe lower layers without coupling to them. The implementation pattern: **SQL views**.

A higher system (e.g. S4) needs derived signals from a lower system (e.g. S1). Two approaches:

1. **Table** — S1 writes to a signals table on every event. Couples S1 to S4: S1 must know what S4 needs. Inverts the hierarchy.
2. **View** — S4 defines a view over S1's tables. S1 captures facts without knowing S4 exists. S4 reads through its own lens. No coupling.

| Mechanism | Role | Example |
|-----------|------|---------|
| Lower-system tables | Operations data | `observations`, `prompts`, `sessions` |
| Views over those tables | Inter-system channel | `prompt_signals` (S4 reading S1) |
| Higher-system tables | Higher system's conclusions | `work_units` (S4's state) |

The view definition belongs in the consuming system's code, not in shared schema infrastructure. Infrastructure provides the base tables; each system creates its own derived views. This keeps cognitive layers decoupled — each layer can evolve its observation lens independently.

This pattern repeats wherever a higher system interprets lower-system data: S3 observing S1 for retention decisions, S3* auditing S1 for integrity, S5 reading S3 and S4 to mediate policy.

## Platform Constraints as Cognitive Boundaries

The original framing assumed the agent has full control over its context window. Building nmem against Claude Code's hook API revealed that platform constraints define the cognitive boundaries:

| Capability | Available |
|-----------|-----------|
| Observe all tool calls, prompts, session events | Yes — 14 hook events |
| Inject context at session boundaries | Yes — SessionStart `additionalContext` |
| Inject context mid-session | Partial — async delivery, multiple bugs |
| Clear or edit context programmatically | No |
| Access token counts or cost data | No — API-only |

**Implication:** nmem has eyes but no hands. It can observe everything but can only act at session boundaries. This constraint shaped S4's design more than any theoretical consideration — S4 operates reactively (detect, store, inject on next SessionStart) rather than autonomously (detect, clear, re-inject mid-session). Full autonomous context control requires either upstream hook improvements or an API-based agent harness that bypasses Claude Code.

The cognitive architecture adapts to its platform the way biological cognition adapts to its substrate. The constraints aren't limitations to overcome — they're the environment the system lives in.

## Design Implications

1. **Prompts are more valuable than observations.** Confirmed. The `prompt_id` FK is the most important schema decision. Intent is harder to reconstruct than action.

2. **Reasoning blocks are the highest-signal data source.** Confirmed. They contain 84% of content volume but also the richest reasoning signal. The `learned` field in summaries draws primarily from thinking blocks.

3. **Outcomes matter more than perception.** Revised. The original "discard tool responses" was too aggressive. Failure outcomes are high signal. Success outcomes can be inferred from temporal patterns (no failure following an action).

4. **Compression is a day-one concern at real volumes.** Revised. The 5 MB/year estimate was off by 100x. At ~652 MB/year, session summarization is essential infrastructure, not future optimization.

5. **Each cognitive layer is a view over the data below it.** Confirmed and strengthened. This throwaway line became architecturally load-bearing. SQL views as inter-system channels enforce VSM hierarchy: higher systems observe, lower systems don't change.

6. **The schema supports recursive cognition, not linear levels.** New. S1's S4 (within-session compression) and the outer S4 (cross-session synthesis) operate on the same data at different granularities. The schema must support both without either knowing about the other.

7. **Platform constraints shape cognitive architecture.** New. The available actuators (not just sensors) determine what kind of cognition is possible. Design for the platform you have, not the one you want.

## Retrieval as Cognition

The queries you run against memory are themselves cognitive acts. "What files did I edit on nmem?" is a memory retrieval. "What was I thinking when I decided to store everything?" is episodic recall. "Find sessions similar to this one" is pattern matching.

The quality of these queries determines the quality of the memory system. A schema that supports rich queries enables richer cognition. FTS5, intent joins, session signatures, and cross-project queries are not features — they're cognitive capabilities.

The signal multiplication insight extends this: the richest queries *join* across cognitive layers. Current hot files (micro) × past learned decisions (macro) = relevance-ordered retrieval. The join is the cognition.
