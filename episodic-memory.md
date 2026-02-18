# Episodic Memory — Cognitive Science Foundations for Agent Memory Systems

Theory and design principles from cognitive science research on episodic memory, event segmentation, and narrative memory. Applied to agent memory systems that need to encode, segment, store, and retrieve bounded experiences from continuous interaction.

## Core Distinction: Episodic vs Semantic Memory

Endel Tulving (1972, 1983) established the foundational distinction:

| | Episodic Memory | Semantic Memory |
|---|---|---|
| **Content** | Specific experiences bound to time and context | General knowledge, facts, concepts |
| **Encoding** | Single-shot — one experience is enough | Gradual — abstracted from many exposures |
| **Retrieval** | Re-experiencing (mental time travel) | Knowing (factual recall) |
| **Context** | Rich — when, where, why, what happened | Decontextualized — the fact without the story |
| **Example** | "I debugged the auth bug yesterday by finding a stale mock" | "Stale mocks cause test failures" |

The distinction matters for agent memory: session summaries and learnings are *semantic* (abstracted knowledge). Episodes are *episodic* (specific experiences with narrative structure). Both are needed. Episodic memory is richer — it preserves causality, sequence, and context that semantic compression loses.

### The SPI Model (Tulving)

Information flows: **S**erial encoding → **P**arallel storage → **I**ndependent retrieval.

- Encoding is serial: perceptual input → semantic interpretation → episodic binding. You must understand what happened (semantic) before you can remember it as an experience (episodic).
- Storage is parallel: episodic and semantic traces coexist.
- Retrieval is independent: you can recall the experience without the fact, or the fact without the experience.

**Design implication**: An agent memory system should encode observations semantically first (classify, extract, structure) then bind them into episodes (temporal context, intent, narrative). Retrieval can target either layer independently.

### Semanticization — Episodes Become Knowledge

Episodic memories gradually lose contextual detail and become semantic knowledge over time. "I fixed the auth bug by updating the mock" → "stale mocks cause test failures" → implicit knowledge that you check mocks when tests fail.

This is not decay — it is compression. The specific experience transforms into general knowledge that applies across contexts. The navigational theory proposes that repeated exposure to similar episodes accelerates this transformation.

**Design implication**: Episode summaries are the intermediate form between raw experience and durable knowledge. `nmem learn` (cross-session pattern detection) is a semanticization process — it detects recurring episodes and produces abstract patterns. The episodic-to-semantic pipeline: raw observations → episode narratives → cross-session patterns → durable learnings (CLAUDE.md).

## Event Segmentation Theory (EST)

How does continuous experience get divided into discrete episodes? Event Segmentation Theory (Zacks et al., 2007) provides the dominant framework.

### The Prediction Error Model

The brain maintains an **event model** — a working memory representation of the current event. This model generates predictions about what will happen next. When prediction error spikes (what happens diverges from what was expected), the event model is updated — this is an **event boundary**.

```
... ongoing activity → prediction matches → same event ...
... prediction error spike → event model update → new event boundary ...
... new event model → predictions resume → same event ...
```

Event boundaries are not arbitrary segmentation. They occur at points of genuine discontinuity — when the current model of what's happening stops being adequate.

### Bayesian Surprise vs Simple Error

Kumar et al. (2023) applied this to story listening using GPT-2 to compute prediction distributions. Key finding: **Bayesian surprise** (how much the entire probability distribution shifted) predicts event boundaries better than simple surprisal (how unlikely a single word was).

The distinction matters: encountering an unexpected word after confident prediction is a stronger boundary signal than encountering an unlikely word when predictions were already uncertain. It's the *shift in confidence*, not the raw error, that signals a new event.

**Design implication**: For prompt-driven episode detection, the boundary signal is not "did the keywords change?" (simple surprisal) but "did the intent shift from a stable state?" A user refining the same task with slightly different vocabulary is not a boundary. A user confidently pivoting to a new topic is.

### SEM — Structured Event Memory (Franklin et al., 2020)

The Structured Event Memory model combines symbolic and neural approaches:

1. **Event schemas** — learned templates for recurring event types. "Debugging session" is a schema: investigation → hypothesis → fix → test → resolution.
2. **Schema inference** — the model infers which schema is active, and detects when a schema switch occurs (event boundary).
3. **Schema-based reconstruction** — memories are recalled using the schema as scaffolding. You remember the specific experience through the lens of the general pattern.

SEM uses nonparametric Bayesian inference: the number of schemas is not fixed. New event types create new schemas. Familiar event types are recognized and mapped to existing schemas.

**Design implication**: Episodes in an agent system follow recognizable patterns (debug cycle, feature implementation, refactoring, design discussion). These patterns are schemas. Detecting that the current episode matches a known schema enables richer narrative construction — the story has a recognized arc, not just a sequence of events.

## Episodes as Stories

An episode is not a time range with metadata. It is a self-contained narrative.

### Narrative Structure

Every episode has:

- **Beginning** — the inciting intent. What are we trying to do and why? In an agent system, this is the user's directive prompt.
- **Middle** — the unfolding action. Investigation, execution, failure, recovery. The dialogue between user and agent, the tools invoked, the results observed. This is the body of the story.
- **End** — resolution. Success (committed, deployed), failure (abandoned, deferred), or interruption (session ended, intent shifted). The resolution determines the episode's meaning.

### What Makes a Story Memorable

Cognitive research identifies properties that make episodes stick:

- **Causal coherence** — events are connected by cause and effect, not just sequence. "The test failed *because* the mock was stale" is more memorable than "the test failed, then the mock was updated."
- **Emotional/motivational valence** — episodes involving struggle, failure, surprise, or breakthrough are remembered better than routine success. Failures carry stronger signal than successes.
- **Distinctiveness** — episodes that differ from the schema are better remembered. A debug session that followed the normal arc is forgotten; one where the "fix" made things worse is remembered.
- **Goal relevance** — episodes connected to active goals are prioritized in retrieval. Memory serves current needs.

**Design implication**: Episode summaries should capture causality (not just sequence), highlight failures and surprises (not just completions), and note deviations from expected patterns. A summary that says "edited 3 files, ran tests, committed" is worse than "tried patching the handler directly, tests broke because the mock was stale — had to update the mock first, then the fix worked."

## Five Properties of Episodic Memory for Agent Systems

From the position paper "Episodic Memory is the Missing Piece for Long-Term LLM Agents" (Feng et al., 2025):

| Property | Meaning | Agent Implementation |
|----------|---------|---------------------|
| **Long-term storage** | Persists across sessions | External store (SQLite), not just context window |
| **Explicit reasoning** | Can reflect on and reason about memories | Searchable, retrievable episodes with narrative content |
| **Single-shot learning** | One experience is enough to encode | Each episode is stored from a single occurrence |
| **Instance-specific** | Details unique to a particular occurrence | Specific files, commands, errors — not abstracted |
| **Contextual relations** | When, where, why, how bound together | Intent + observations + outcome in one unit |

Working memory (context window) has all properties except long-term storage. Semantic memory (learnings, CLAUDE.md) has long-term storage and explicit reasoning but lacks instance-specificity and single-shot encoding. Only episodic memory has all five.

## Encoding Specificity Principle

Tulving's encoding specificity principle: memory retrieval depends on the match between encoding context and retrieval context. What you stored the memory *with* determines what cues will retrieve it.

Forgetting may not be decay — it may be the absence of appropriate retrieval cues. The memory exists but can't be found because the current context doesn't match the encoding context.

**Design implication**: Episodes should be indexed by their encoding context — the intent, the files involved, the error patterns, the project. When the next session encounters a similar context (same files, same error, same intent), the matching episode should surface. This is not keyword search — it's context matching. The retrieval cue is the current situation, and the memory surfaces because its encoding context overlaps with the present.

## Memory Consolidation Pipeline

The full pipeline from experience to durable knowledge:

```
Raw experience (tool calls, prompts, responses)
    ↓ event segmentation (boundary detection)
Episodes (bounded narratives with beginning/middle/end)
    ↓ narrative construction (LLM summarization)
Episode summaries (compressed stories preserving causality)
    ↓ cross-session pattern detection (semanticization)
Patterns (recurring episodes, stuck loops, repeated failures)
    ↓ collaborative review + integration
Durable knowledge (CLAUDE.md, learnings)
```

Each stage compresses and abstracts. Raw observations are too detailed. Episode summaries preserve causality at the cost of detail. Patterns lose individual episodes but capture recurring structure. Durable knowledge loses everything except the conclusion.

The episodic layer (episode summaries) is the critical middle ground — compressed enough to be useful, specific enough to preserve the story. Without it, you jump from raw observations to abstract knowledge and lose the narrative that connects them.

## Retrieval: How Episodes Are Recalled

### Cue-Dependent Retrieval

Episodes are retrieved by matching the current context against stored encoding contexts. Multiple cue types:

- **Intent overlap** — current task resembles a past episode's goal
- **File overlap** — working on files that appeared in a past episode
- **Error pattern match** — encountering an error seen in a past episode
- **Schema match** — recognizing a familiar event type (debug cycle, refactoring)

The best retrieval uses multiple cues simultaneously (composite scoring), matching the multi-dimensional encoding context of the original episode.

### Interference and Discrimination

When many similar episodes exist, retrieval becomes harder (proactive interference). The system must discriminate between similar but distinct episodes. Distinctive features — what made *this* episode different from the usual pattern — are the key.

**Design implication**: Episode summaries should foreground what was distinctive, not what was routine. "Ran tests, they passed" is schema-default and not worth storing. "Tests failed because the PATH was missing in dispatched tmux sessions" is distinctive and retrievable.

## Relevance to nmem ADR-010

The cognitive science maps directly to nmem's design:

| Cognitive Concept | nmem Mapping |
|---|---|
| Event Segmentation (EST) | Boundary detection via prompt-driven intent shift |
| Event model / prediction error | User prompt introduces new intent → boundary |
| Terse prompts as continuations | Low prediction error → same event model |
| Event schemas (SEM) | Recognizable episode patterns (debug, implement, design) |
| Narrative structure | Episode as story: beginning (intent) → middle (dialogue + observations) → end (resolution) |
| Episodic → semantic pipeline | Episodes → `nmem learn` patterns → CLAUDE.md |
| Encoding specificity | Retrieval by context match (intent, files, errors) |
| Bayesian surprise | Intent shift from stable state, not just keyword change |
| Semanticization | Episodes gradually lose detail, become general knowledge |
| Signal multiplication | Episode × cross-session pattern = actionable (stuck loop warning) |

## Key Sources

- Tulving, E. (1983). *Elements of Episodic Memory*. Oxford University Press.
- Tulving, E. (2002). Episodic memory: from mind to brain. *Annual Review of Psychology*, 53, 1–25.
- Zacks, J.M., et al. (2007). Event perception: A mind-brain perspective. *Psychological Bulletin*, 133(2), 273–293.
- Franklin, N.T., et al. (2020). Structured Event Memory: A neuro-symbolic model of event cognition. *Psychological Review*, 127(3), 327–361.
- Kumar, M., et al. (2023). Bayesian Surprise Predicts Human Event Segmentation in Story Listening. *Cognitive Science*, 47(10).
- Feng, G., et al. (2025). Position: Episodic Memory is the Missing Piece for Long-Term LLM Agents. *arXiv:2502.06975*.
- Reynolds, J.R., Zacks, J.M., & Braver, T.S. (2007). A computational model of event segmentation from perceptual prediction. *Cognitive Science*, 31(4), 613–643.
- Greenberg, D.L. & Verfaellie, M. (2010). Interdependence of episodic and semantic memory. *Journal of the International Neuropsychological Society*, 16(5), 748–753.
