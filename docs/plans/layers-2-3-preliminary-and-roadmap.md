# Layers 2-3 Preliminary Analysis and Overall Roadmap

**Date:** 2026-02-07
**Status:** Preliminary — to be revised after Layer 1 implementation
**Prerequisite:** `layer-1-gap-analysis.md` (completed)

---

## How to Read This Document

This document captures what we currently know and think about Layers 2 and 3 of the Attractor nlspec as they relate to Amplifier. It is written **open-handed** — we don't know what we don't know, and we expect this analysis to change substantially as we work.

The approach is iterative:

```
Layer 1 analysis (done) -> Layer 1 implementation -> revisit Layer 2 analysis
    -> Layer 2 implementation -> revisit Layer 3 analysis -> Layer 3 implementation
```

Each layer's implementation will reveal things that change our understanding of the next layer. The streaming protocol is the clearest example: we deferred it from Layer 1 because Layer 2's requirements will inform the right design. We expect similar discoveries throughout.

What follows is our best current thinking, marked with confidence levels:
- **High confidence** — grounded in code we've read, unlikely to change
- **Medium confidence** — reasonable inference, may shift with implementation
- **Speculative** — working hypothesis, expect revision

---

## Layer 2: Coding Agent Loop vs Amplifier Orchestrator + Tools

### What the Attractor Spec Defines

The `coding-agent-loop-spec.md` (1,451 lines) defines a **programmable agentic loop** — a library (not a CLI) that pairs an LLM with developer tools. Key characteristics:

- **Provider-aligned toolsets**: Each provider (OpenAI, Anthropic, Gemini) gets its **native** tool definitions and system prompts, byte-for-byte matching the provider's reference agent (codex-rs, Claude Code, gemini-cli)
- **Core loop**: `process_input()` runs: build request -> call LLM -> if tool calls: execute tools -> drain steering -> loop detection -> repeat until no tool calls or limits hit
- **Session state machine**: IDLE -> PROCESSING -> AWAITING_INPUT -> CLOSED
- **Steering**: Inject messages mid-task between tool rounds (not between turns — between iterations within a single turn)
- **Follow-up**: Queue messages for after current input completes
- **Loop detection**: Pattern matching on tool call signatures (window of 10, patterns of length 1-3)
- **Subagents**: Child sessions for parallel/scoped work
- **Execution environment abstraction**: LocalExecutionEnvironment, extensible to Docker, Kubernetes, WASM, SSH

### Initial Mapping to Amplifier (Medium Confidence)

| Attractor Concept | Amplifier Equivalent | Confidence | Notes |
|---|---|---|---|
| Coding agent loop | Orchestrator module | High | The orchestrator IS the loop |
| Provider-aligned toolsets | Tool modules + provider config | Medium | Tools are provider-agnostic in Amplifier today |
| Session state machine | `AmplifierSession` lifecycle | High | Session has init/execute/cleanup states |
| Steering (mid-task injection) | `inject_context` hook action with `ephemeral=True` | Medium | Progress-monitor already does this pattern |
| Follow-up queue | Not directly mapped | Low | May need orchestrator enhancement |
| Loop detection | Not implemented | Low | New capability needed |
| Subagents | `session.spawn` / task tool agent delegation | High | Foundation-layer pattern already exists |
| Execution environment | Tool modules (bash, filesystem) | Medium | Currently hardcoded to local; abstraction possible |
| Tool output truncation | Tool modules handle internally | Medium | Need to verify truncation patterns |
| System prompt layering | Bundle composition + context files | High | Amplifier's bundle system handles this |

### What We Think We'll Find (Speculative)

**Provider-aligned toolsets** is the most interesting divergence. Amplifier's tools are provider-agnostic — `tool-filesystem`, `tool-bash`, etc. work the same regardless of which provider is active. The Attractor spec argues that each provider should get tools that match its reference agent's exact tool definitions (e.g., Anthropic's `edit_file` with `old_string/new_string` vs OpenAI's `apply_patch` with v4a format).

This could go several ways:
1. **We don't need this** — Amplifier's provider-agnostic tools work well enough. The providers translate tool specs to their native format already.
2. **We need provider-specific tool adapters** — A translation layer that presents the same tool capability through different interfaces depending on the active provider.
3. **We need a tool aliasing/mapping system** — The orchestrator maps tool specs based on provider, without changing the underlying tool implementation.

We won't know until we look closely at the spec's tool definitions and compare against our actual tool implementations.

**Steering** is where the `inject_context` action becomes critical. The Attractor's steering concept is: while the agent is in the middle of a tool loop (between LLM call N and LLM call N+1), an external system can inject a message that the agent sees on the next iteration. Amplifier's `inject_context` with `ephemeral=True` does exactly this — `hooks-progress-monitor` already demonstrates the pattern. The question is whether the orchestrator's handling of ephemeral injections is robust enough for the pipeline engine's needs.

**Loop detection** is genuinely new. The spec describes pattern matching on tool call signatures: if the same sequence of tool calls repeats (window of 10, patterns of length 1-3), the agent is stuck. This would likely be a hook module on `tool:post` that tracks call patterns and returns `inject_context` with a "you're looping" warning, or `deny` to break the cycle. The infrastructure supports it; the logic doesn't exist yet.

**Execution environment abstraction** is interesting for the factory use case. Today, tool-bash runs commands on the local machine. For a software factory, you'd want to run in isolated environments (Docker containers, sandboxed filesystems). This maps to a tool module concern — you'd swap `tool-bash-local` for `tool-bash-docker` — but the abstraction might need to be more systematic than ad-hoc tool swapping.

### Areas That May Surprise Us

- **The orchestrator's iteration model** may not match the spec's expectations. Our orchestrators have a `max_iterations` config and a simple while loop. The spec's loop has more nuanced termination conditions, pause/resume semantics, and explicit state transitions.
- **The relationship between orchestrator and provider for streaming** will become clearer. If the coding agent loop needs real-time streaming for responsive tool execution, that changes the streaming protocol design we deferred from Layer 1.
- **The tool execution model** (parallel vs sequential) may matter more than we think. The spec explicitly discusses execution ordering and how tool results flow back to the LLM. Our orchestrators differ on this (loop-basic/streaming: parallel, loop-events: sequential).

### Plan

After Layer 1 implementation is complete:
1. Deep-dive the coding-agent-loop-spec with the same rigor we applied to unified-llm-spec (extract all requirements, map to existing code)
2. Deep-dive our orchestrator and tool modules with fresh eyes (Layer 1 changes may shift what's possible)
3. Determine which orchestrator is the right base to evolve (or whether we need a new one)
4. Determine whether provider-aligned toolsets matter or whether our provider-agnostic approach is sufficient
5. Revise this section with code-grounded findings

---

## Layer 3: Pipeline Engine vs Amplifier ???

### What the Attractor Spec Defines

The `attractor-spec.md` (2,083 lines) defines a **DOT-based pipeline runner** — the crown jewel of the system. Key characteristics:

- **Workflows as graphs**: Directed graphs in Graphviz DOT syntax. Each node is an AI task; edges are transitions. The graph IS the workflow.
- **9 node types**: start, exit, codergen (LLM task), wait.human, conditional, parallel, parallel.fan_in, tool, stack.manager_loop
- **Natural-language edge conditions**: Edges have `condition` expressions and `label` attributes evaluated by the LLM or simple boolean logic
- **5-step edge selection algorithm**: condition-matching -> preferred label -> suggested next IDs -> highest weight -> lexical tiebreak
- **Goal gate convergence**: Critical nodes marked `goal_gate=true` must succeed before the pipeline can exit. If they haven't, the engine jumps to retry targets.
- **Context fidelity**: 6 modes controlling how much context each node sees (full, truncate, compact, summary:low/medium/high)
- **Checkpoint and resume**: JSON checkpoint after every node. Crash recovery from last checkpoint.
- **Model stylesheet**: CSS-like syntax for assigning models per node class
- **Parallel fan-out/fan-in**: Branches with isolated context clones, join policies (wait_all, k_of_n, first_success, quorum)
- **Manager/supervisor loop**: Observe telemetry, evaluate progress, optionally steer child pipeline

### Why This May Not Be Recipes (High Confidence)

Amplifier's recipe system is designed for **linear/staged YAML workflows** with agent handoffs. The Attractor pipeline is fundamentally different:

| Aspect | Amplifier Recipes | Attractor Pipeline |
|---|---|---|
| Structure | Linear steps or staged groups | Arbitrary directed graph |
| Transitions | Sequential (next step) or staged (approval gates) | LLM-evaluated edge conditions with 5-step selection |
| Convergence | None (runs to completion) | Goal gates with retry loops |
| Parallelism | None (sequential execution) | Fan-out/fan-in with join policies |
| Context management | Context accumulation across steps | 6 fidelity modes per node |
| Checkpointing | Session-level | Per-node with crash recovery |
| Visual representation | YAML text | Graphviz DOT (renderable, diffable) |

Recipes could potentially be extended with graph support, but the execution model is different enough that it may be cleaner to build a purpose-built graph runner. This is a design decision we should make deliberately after Layers 1 and 2 are solid.

### Possible Amplifier Approaches (Speculative)

We see several possible directions. These are hypotheses, not recommendations:

**Option A: New orchestrator module — "loop-pipeline"**

Build the pipeline engine as a new orchestrator type. It receives the DOT graph as configuration, walks the graph, and at each node calls the provider (or spawns a sub-session) based on the node's prompt and type. This fits Amplifier's "swap the orchestrator to change behavior" philosophy.

Pros: Fits existing architecture. Leverages provider/hook/tool infrastructure.
Cons: Orchestrator contract assumes single execute() call returning a string. Pipeline needs richer lifecycle.

**Option B: Application-layer graph runner**

Build the pipeline engine as an application (like amplifier-app-cli) that creates and manages AmplifierSessions for each pipeline node. The runner is the orchestrator of orchestrators — it walks the graph and spawns sessions.

Pros: Clean separation. Each node gets its own session with full lifecycle.
Cons: Heavier than needed if nodes are simple LLM calls.

**Option C: Extend recipes with graph support**

Add graph primitives to the recipe system: conditional edges, convergence loops, fan-out/fan-in. Recipes become a superset of both linear and graph workflows.

Pros: One system for all workflows.
Cons: May contort the recipe system beyond its design intent.

**Option D: Hybrid — thin graph runner that delegates to existing primitives**

A lightweight graph walker that uses:
- `session.execute()` for codergen nodes (LLM tasks)
- `session.spawn()` for subagent/parallel nodes
- Hooks for human gates (`ask_user`)
- Hooks for context injection between stages (`inject_context`)
- Recipes for sub-workflows within a node

Pros: Maximum reuse of existing infrastructure.
Cons: Coordination complexity.

We genuinely don't know which is right yet. Layer 1 and 2 implementation will clarify what primitives we have and what's missing.

### Specific Concepts to Think About

**Context fidelity** is one of the most interesting concepts in the spec. The idea that each pipeline node should see a controlled amount of context (not the full conversation history) is powerful for multi-stage pipelines where context windows are finite. Amplifier's context modules handle message history, but they don't have a "fidelity mode" concept. This might map to:
- Different context module configurations per session (if using Option B)
- Context compaction hooks (the context system already supports compaction)
- A pre-provider hook that summarizes/truncates context based on node config

**Goal gate convergence** is the pipeline's answer to "how do you know the software is done?" Nodes marked as goal gates must succeed. If the pipeline reaches the exit without satisfying all goal gates, it retries. This is fundamentally a graph execution concept — it requires the runner to track node outcomes and backtrack. None of our existing primitives directly support this, though the while/convergence loops in recipes are a distant relative.

**Model stylesheet** (`*.code { llm_model: claude-sonnet-4-5; }`) is elegant. In Amplifier terms, this could be a hook on `provider:request` that reads the pipeline node's class attribute and applies model routing via `modify`. After Layer 1 Phase 3 (orchestrator consistency + ChatRequest.model field), the infrastructure would support this.

**DOT syntax** as the workflow definition format is a deliberate choice. It's renderable (instant visual feedback), diffable (version-controllable), and a natural fit for graph structures. If we build a pipeline engine, we'd want to support DOT input. There are Python DOT parsers (pydot, graphviz) that could handle this.

### What We Need From Layers 1 and 2 First

Before we can make real decisions about Layer 3:

| Dependency | Why |
|---|---|
| Layer 1 error taxonomy | Pipeline needs to catch provider errors and decide retry vs fail |
| Layer 1 ChatRequest.model | Pipeline nodes need per-node model selection |
| Layer 1 Usage fields | Pipeline needs cost tracking across nodes |
| Layer 2 streaming decision | Pipeline nodes may need streaming output |
| Layer 2 steering/injection | Pipeline needs to inject context between stages |
| Layer 2 execution environment | Pipeline's parallel branches may need isolated environments |
| Layer 2 orchestrator evolution | Determines whether pipeline nodes use existing orchestrators or something new |

---

## Overall Roadmap

### The Iterative Pattern

```
Analyze Layer N -> Implement Layer N -> Discoveries inform Layer N+1 analysis
                                              |
                                     Revise Layer N+1 plan
                                              |
                                     Implement Layer N+1
                                              |
                                           ... repeat
```

We commit to the current layer's implementation. We hold the next layer's analysis loosely.

### Phase Map

```
LAYER 1: Unified LLM Client                          NOW
===========================================================
Phase 1: Kernel vocabulary (~65 lines)              ------>
  - Usage fields (reasoning, cache tokens)
  - ChatRequest fields (model, tool_choice, stop, etc.)
  - Error taxonomy

Phase 2: Provider improvements                      ------>
  - Shared retry utility (extract from Anthropic)
  - Error translation in all 3 providers
  - Cache/reasoning token surfacing
  - Consistent thinking trigger

Phase 3: Orchestrator consistency                   ------>
  - Consistent provider:request/response events
  - Pass ChatRequest fields through
  - Hook-driven model routing

Phase 4: Module graduation                          ------>
  - Cost tracker module
  - Scheduler hooks for provider-level routing

    |
    v  (Layer 1 complete. Revisit Layer 2 analysis.)

LAYER 2: Coding Agent Loop                          NEXT
===========================================================
Phase 5: Deep analysis with code inspection         
  - Map coding-agent-loop-spec to existing orchestrators
  - Evaluate provider-aligned toolsets vs our approach
  - Assess steering/injection capabilities
  - Determine loop detection approach

Phase 6: Orchestrator evolution                     
  - Evolve existing orchestrator(s) or build new one
  - Add loop detection
  - Add steering/follow-up queue
  - Resolve streaming protocol (deferred from Layer 1)

Phase 7: Tool and environment improvements          
  - Tool output truncation standardization
  - Execution environment abstraction (if needed)
  - System prompt layering improvements (if needed)

    |
    v  (Layer 2 complete. Revisit Layer 3 analysis.)

LAYER 3: Pipeline Engine                            FUTURE
===========================================================
Phase 8: Architecture decision                      
  - Choose approach (new orchestrator, app-layer, recipe extension, hybrid)
  - Design graph runner with DOT input
  - Design context fidelity system
  - Design goal gate convergence

Phase 9: Core pipeline implementation               
  - Graph parsing and validation
  - Node execution (delegating to Layer 1+2 primitives)
  - Edge selection algorithm
  - Checkpoint and resume

Phase 10: Advanced pipeline features                
  - Parallel fan-out/fan-in
  - Manager/supervisor loops
  - Model stylesheet
  - Human-in-the-loop gates
  - HTTP server mode (if needed)
```

### What We're Committing To vs Holding Loosely

| Item | Commitment Level |
|---|---|
| Layer 1 gap analysis | **Committed** — grounded in 17 repos of actual code |
| Layer 1 implementation plan | **Committed** — specific files, fields, effort estimates |
| Layer 2 mapping | **Held loosely** — reasonable inference, will revise |
| Layer 2 implementation plan | **Not committed** — depends on Layer 1 discoveries |
| Layer 3 approach | **Speculative** — multiple options, decision deferred |
| Layer 3 implementation | **Not committed** — depends on Layer 1 + 2 |

### Decision Points

These are moments where we'll pause and make explicit choices:

1. **After Layer 1 Phase 1 (kernel):** Do the new fields work as expected? Do any providers need adjustment to the field names or types?

2. **After Layer 1 Phase 2 (providers):** Does the shared retry utility work across all 3 providers? Does error translation surface any unexpected behaviors?

3. **After Layer 1 Phase 3 (orchestrators):** Does consistent event emission reveal any performance concerns? Does hook-driven model routing work in practice?

4. **After Layer 2 analysis (deep dive):** Which orchestrator is the right base? Do we need provider-aligned toolsets? What streaming contract does the pipeline need?

5. **After Layer 2 implementation:** What primitives do we now have? What's missing for Layer 3? Which pipeline approach (A/B/C/D) fits best?

6. **After Layer 3 architecture decision:** Build vs extend? New module type? Application layer?

### Success Criteria

**Layer 1 is successful when:**
- All 3 providers translate errors to kernel vocabulary
- All 3 providers report reasoning and cache tokens in Usage
- All 3 orchestrators emit provider:request and provider:response consistently
- A hook on provider:request can modify the model for any orchestrator
- Existing tests pass. No caller breaks.

**Layer 2 is successful when:**
- An Amplifier session can run the equivalent of the coding-agent-loop-spec's core loop
- Steering (mid-task injection) works for an external pipeline engine
- Loop detection prevents stuck agents
- The streaming question is resolved

**Layer 3 is successful when:**
- A DOT graph can be parsed and executed as a pipeline
- Goal gates enforce convergence
- Pipeline nodes can use different models via stylesheet
- Human gates work via the existing hook/approval system
- Checkpoints enable crash recovery

**The whole thing is successful when:**
- The Attractor nlspec can be implemented on top of Amplifier's primitives
- No Amplifier philosophy is compromised (mechanism not policy, bricks and studs, ruthless simplicity)
- Existing Amplifier users are unaffected by the additions

---

## Open Questions

These are things we genuinely don't know yet. We're listing them so we remember to answer them as we go.

### Layer 2 Open Questions
- Does provider-aligned toolset matter in practice, or is our provider-agnostic approach sufficient? (The Attractor spec is opinionated about this, but it's testable.)
- What does the steering/injection contract need to look like for the pipeline engine to use it? Is `inject_context` with `ephemeral=True` enough, or do we need a more structured API?
- Should loop detection be a hook module or an orchestrator feature? (Hook is more composable; orchestrator is more integrated.)
- Does the coding agent loop need its own session state machine, or does `AmplifierSession`'s lifecycle suffice?
- How does the execution environment abstraction relate to Amplifier's tool module system? Is it a tool concern or a session concern?

### Layer 3 Open Questions
- Is the pipeline engine an orchestrator, an application, a recipe extension, or something new?
- How does context fidelity map to Amplifier's context module system?
- Can goal gate convergence be built on top of existing while/convergence loop primitives in recipes, or does it need purpose-built graph traversal?
- Does the pipeline engine need to be a single process, or can nodes run as separate sessions/processes?
- What's the right checkpoint format? Is it compatible with Amplifier's existing session persistence?
- How does the model stylesheet interact with hook-driven model routing from Layer 1?

### Cross-Cutting Open Questions
- Will we want a thin `generate()` convenience function that wraps Provider + Orchestrator + Hooks into a single call? If so, where does it live?
- How does the pipeline engine interact with the Amplifier CLI? Is it a new command (`amplifier pipeline run graph.dot`)?
- Should the pipeline engine be an Amplifier module, a bundle, or a standalone application?
- How do we test pipelines? The Attractor spec has a "Definition of Done" with cross-feature parity matrices. What's our equivalent?

---

## What We Expect to Learn

Based on experience with Layer 1, here's what we think each layer will teach us:

**Layer 1 implementation will likely reveal:**
- Whether `extra="allow"` is truly sufficient for provider-specific features, or whether some things need explicit kernel fields
- Whether the orchestrator inconsistencies (parallel vs sequential tools, different event emission) are deeper than surface-level
- Whether the retry utility works cleanly across all 3 providers' different error shapes

**Layer 2 analysis will likely reveal:**
- Whether our orchestrators need fundamental evolution or just incremental improvement
- What streaming contract the pipeline engine actually needs (resolving the Layer 1 deferral)
- Whether provider-aligned toolsets are a real need or an optimization we can skip

**Layer 3 analysis will likely reveal:**
- Which of the four approaches (new orchestrator, app-layer, recipe extension, hybrid) is the natural fit
- Whether Amplifier's philosophy (mechanism not policy) supports a graph runner, or whether the graph runner IS the policy layer
- What new kernel primitives (if any) are needed for graph-structured execution

We'll update this document as each layer teaches us something new.
