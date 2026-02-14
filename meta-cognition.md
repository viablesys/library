# Meta-Cognition — Thinking About Thinking in Agent Systems

How an agent system observes, records, and reasons about its own cognitive processes. Applied to memory systems (nmem), hook-based observation, and session continuity.

## Core Concepts

### Cognitive Loop

An agent operates in a loop: receive intent → reason → act → observe result → reason again. Each stage produces artifacts:

| Stage | Artifact | nmem Mapping |
|-------|----------|-------------|
| Receive intent | User prompt | `prompts` (source=user) |
| Reason | Thinking block | `prompts` (source=agent) |
| Act | Tool call | `observations` |
| Observe result | Tool response | Discarded (raw payload) |
| Reason again | Next thinking block | `prompts` (source=agent) |

The memory system captures the *intent* (why) and the *action* (what), but intentionally discards the *perception* (tool responses — file contents, command output). The perception is ephemeral; the intent and action are the durable signal.

### Levels of Cognition

**Level 0 — Reactive.** Execute tool calls. No memory. Each session starts cold. This is Claude Code without nmem.

**Level 1 — Observational.** Record what happened. Session transcripts, capture.jsonl. Raw data, no interpretation. Useful for replay, not for reasoning.

**Level 2 — Structured.** Extract meaning from observations. Classify events (file_read, command, search). Link actions to intent (prompt_id). Compute session signatures. This is nmem's current design target.

**Level 3 — Reflective.** Reason about patterns across sessions. "I keep reading this file but never editing it — why?" "Sessions on this project are getting shorter — is the design stabilizing?" "The last three debugging sessions all touched the same module." This requires queries over accumulated structured data.

**Level 4 — Anticipatory.** Predict what context will be needed. "This looks like a refactoring session, so inject past refactoring decisions." "The user mentioned ADR-003, so preload related ADRs." Session signatures + retrieval enable this without explicit planning.

### Self-Referential Observation

nmem observes its own design process. The capture tool records hook events from sessions where the capture tool itself was being built and refined. The prototype DB contains observations about creating the prototype DB. This isn't a bug — it's a feature. The system's design is validated against its own usage data.

This creates a productive recursion:
1. Build a capture tool → generates data about how the tool was built
2. Analyze that data → reveals patterns that inform the schema
3. Build a schema → tested against the data from step 1
4. Schema captures future sessions → including sessions where the schema is refined

The recursion grounds design decisions in real data rather than hypothetical requirements.

### Intent Hierarchy

Not all intents are equal. They form a hierarchy:

- **Strategic**: "Let's build nmem" (project-level, spans sessions)
- **Tactical**: "Add session signatures to ADR-003" (task-level, spans a work unit)
- **Operational**: "Read the file first" (step-level, spans one tool call)

User prompts tend toward tactical. Agent reasoning tends toward operational. The strategic level is implicit — derived from the accumulation of tactical decisions across sessions, not from any single prompt.

nmem's `prompts` table captures tactical and operational intent. Strategic intent emerges from retrieval patterns: "What have I been working on for the last week?" is a strategic query answered by aggregating tactical observations.

### Forgetting as Cognition

A memory system that stores everything has perfect recall but no understanding. Understanding requires forgetting — or more precisely, *compression*. The dedup strategy in ADR-003 is a form of compression: collapsing repeated reads into a single observation. Session signatures are compression: reducing 200 observations to a 5-element vector.

The progression: raw events → structured observations → session summaries → project-level patterns. Each level compresses the one below. The lossy steps are where judgment happens — what to keep, what to collapse, what to discard. This is meta-cognition: the system deciding what's worth remembering.

### Cross-Session Identity

Without persistent memory, each session is a new agent. It has access to code and docs but no experiential continuity — it doesn't know what it tried yesterday, what worked, what failed.

nmem's SessionStart context injection bridges this gap. The injected context says: "Here's what you (a previous instance of yourself) did on this project. Here's what you were thinking. Here are the files you keep returning to." This creates a form of identity continuity — not full consciousness, but enough to avoid repeating mistakes and build on prior work.

The quality of this continuity depends entirely on what gets injected. Too much context and the agent drowns in noise. Too little and it starts cold. The right amount is: recent intent (what was I doing?), key files (what matters?), and unresolved questions (what's still open?).

## Design Implications for nmem

1. **Prompts are more valuable than observations.** The "why" (intent) is harder to reconstruct than the "what" (action). Actions can be inferred from code diffs; intent cannot be inferred from anything except the original prompt or reasoning.

2. **Reasoning blocks are the highest-signal data source.** They contain interpreted intent — not just "what the user said" but "what the user meant, and how I plan to accomplish it." They bridge the gap between user directive and tool execution.

3. **Session signatures enable Level 4 cognition.** Classifying session type from event distribution allows anticipatory context loading without explicit user input.

4. **The schema should support all four levels.** Level 1 data (raw events) feeds Level 2 (structured observations). Level 2 feeds Level 3 (cross-session queries). Level 3 feeds Level 4 (anticipatory injection). Each level is a view over the same underlying data.

5. **Compression is a future concern, not a launch concern.** At nmem's volume (5 MB/year), store everything. Compression (session summaries, observation merging) becomes relevant at scale or when retrieval quality degrades — not before.

## Retrieval as Cognition

The queries you run against memory are themselves cognitive acts. "What files did I edit on nmem?" is a memory retrieval. "What was I thinking when I decided to store everything?" is episodic recall. "Find sessions similar to this one" is pattern matching.

The quality of these queries determines the quality of the memory system. A schema that supports rich queries enables richer cognition. FTS5, intent joins, session signatures, and cross-project queries are not features — they're cognitive capabilities.
