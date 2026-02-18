# Attractor Remaining Gaps: Prioritized Action Plan

> **Execution:** Use the subagent-driven-development workflow to implement this plan.

**Goal:** Close all remaining gaps between the Attractor nlspec and the current implementation, wire up all "built but unwired" infrastructure, implement the three CRITICALs, and validate the full stack end-to-end in a shadow environment.

**Architecture:** The bundle at `amplifier-bundle-attractor/` contains 5 modules (loop-agent, loop-pipeline, tool-apply-patch, tool-report-outcome, hooks-tool-truncation) plus profile bundles and system prompts. All code changes happen inside this bundle repo.

**Tech Stack:** Python 3.11+, Amplifier module system, existing Amplifier tools and providers (all Phase 2 PRs merged).

---

## Gap Verification Summary

After reading every implementation file against the specs, here is the verified state of each gap:

### Confirmed "Built But Unwired" (code exists, just needs integration)

| Gap | Module Built In | What Exists | What's Missing |
|-----|----------------|-------------|----------------|
| GAP-AL-04 | `system_prompt.py` | `build_system_prompt()` with 5-layer assembly | Never called from `agent_session.py` — session sends raw history without a system message |
| GAP-AL-09 | `environment.py` | `build_environment_context()` with git/platform/model info | Never called from `agent_session.py` or `__init__.py` |
| GAP-AL-07 | `system_prompt.py` | `discover_project_docs()` with provider-aware loading | Never called — depends on AL-04 and AL-09 being wired first |
| GAP-PL-06 | `fidelity.py` | `resolve_fidelity()`, `resolve_thread_key()`, `build_preamble()` | Never called from `backend.py` — every spawn creates a fresh session with no preamble |
| GAP-PL-08 | `artifacts.py` | `ArtifactStore` with file-backing, store/get/list | Never instantiated in `engine.py` — `_write_manifest()` creates `artifacts/` dir but no store |
| GAP-PL-09 | `fidelity.py` | Thread key resolution for session reuse | `backend.py` creates a fresh session per node — no session pool or thread-keyed reuse |

### Confirmed "Needs New Code"

| Gap | What's Needed |
|-----|--------------|
| GAP-AL-01 (CRITICAL) | Streaming path in `agent_session.py` — currently only `provider.complete()`, no `provider.stream()` |
| GAP-AL-03 (CRITICAL) | Provider-aligned tool presentation — orchestrator doesn't read profile config or adapt tools |
| GAP-PL-01 (CRITICAL) | `ManagerLoopHandler` is an explicit stub returning SUCCESS immediately |
| GAP-AL-02 | Interactive subagent tools (spawn_agent/send_input/wait/close_agent) — tool-delegate only supports spawn-and-block |
| GAP-PL-07 | `k_of_n` and `quorum` join policies, `fail_fast` and `ignore` error policies in parallel handler |

### Confirmed "Verify Only" (may already work, needs test proof)

| Gap | What To Verify |
|-----|---------------|
| GAP-AL-05 | hooks-tool-truncation registers on `tool:post` — verify the hook event routing connects to `agent_session._execute_single_tool()` |
| GAP-AL-10 | Follow-up queue recursive processing — verify it works with real multi-turn scenarios |
| GAP-AL-11 | Context window warning — verify it fires correctly with realistic message sizes |
| GAP-AL-13 | Anthropic shell timeout 120s — verify profile config flows through to tool-bash |

---

## Critical Path & Dependency Graph

```
Sprint 1 (Wire the Unwired) ─────────────────────────┐
Sprint 2a (Streaming) ──────────────────┐             │
Sprint 2b (Provider-Aligned Tools) ─────┤             │
Sprint 2c (Manager Loop) ──────────────┐│             │
                                       ││             │
                                       ├┴─→ Sprint 3 (HIGH gaps)
                                       │         │
                                       │         ▼
                                       └──→ Sprint 4 (Integration Testing)
                                                  │
                                                  ▼
                                             Sprint 5 (MEDIUM gaps, ongoing)
```

**Parallelizable:** Sprint 1, Sprint 2a, Sprint 2b, Sprint 2c are all independent.
**Sequential:** Sprint 3 needs Sprint 1 + Sprint 2. Sprint 4 needs Sprint 3. Sprint 5 is ongoing.

---

## Sprint 1: Wire the Unwired

**Goal:** All existing infrastructure code is connected to the runtime paths. Fastest wins — code exists, just needs integration calls and tests proving the wiring works.

**Estimated effort:** ~2 hours total (7 tasks, 10-20 min each)

---

### Task 1.1: Wire System Prompt Assembly into Agent Session

**Files:**
- Modify: `modules/loop-agent/amplifier_module_loop_agent/agent_session.py`
- Modify: `modules/loop-agent/amplifier_module_loop_agent/__init__.py`
- Test: `modules/loop-agent/tests/test_system_prompt_wiring.py`

**Context:** `agent_session.py` builds `ChatRequest` with only conversation history messages (line 136-143). The spec (PROV-002) says the system prompt must be rebuilt every LLM call with 5 layers: base prompt, environment, tool descriptions, project docs, user override. `system_prompt.py` and `environment.py` implement this but are never called.

**Step 1: Write the failing test**

```python
# tests/test_system_prompt_wiring.py
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from amplifier_module_loop_agent.agent_session import AgentSession
from amplifier_module_loop_agent.config import SessionConfig

@pytest.mark.asyncio
async def test_system_prompt_included_in_chat_request():
    """The first message in every ChatRequest must be a system message."""
    config = SessionConfig.from_dict({
        "system_prompt": "You are a coding agent.",
        "max_tool_rounds_per_input": 1,
    })
    provider = AsyncMock()
    provider.complete = AsyncMock(return_value=_make_text_response("done"))
    hooks = AsyncMock()
    hooks.emit = AsyncMock()
    tools = {}

    session = AgentSession(config=config, provider=provider, tools=tools, hooks=hooks)
    await session.process_input("hello")

    # Verify provider.complete was called with a system message first
    call_args = provider.complete.call_args
    request = call_args[0][0]
    assert request.messages[0].role == "system"
    assert "You are a coding agent." in request.messages[0].content
```

**Step 2: Run test — expect FAIL**

```bash
cd modules/loop-agent && uv run pytest tests/test_system_prompt_wiring.py::test_system_prompt_included_in_chat_request -v
```
Expected: FAIL — system message not present.

**Step 3: Implement the wiring**

In `agent_session.py`, add the system prompt assembly:

1. Import `build_system_prompt`, `discover_project_docs` from `.system_prompt` and `build_environment_context` from `.environment`.
2. Add `_provider_name` and `_model` fields to `__init__` (passed from `AgentOrchestrator.execute()`).
3. Add `_build_system_prompt()` method that calls the 5-layer assembly.
4. In `process_input()`, before building `ChatRequest`, call `_build_system_prompt()` and prepend a system `Message` to the messages list.

In `__init__.py`, pass provider name and model info when constructing `AgentSession`:
- Extract provider name from `providers` dict key.
- Extract model from config or provider info.

**Step 4: Run test — expect PASS**

```bash
cd modules/loop-agent && uv run pytest tests/test_system_prompt_wiring.py -v
```

**Step 5: Commit**

```bash
git add modules/loop-agent/
git commit -m "feat(loop-agent): wire system prompt assembly into agent session"
```

---

### Task 1.2: Wire Environment Context into Agent Session

**Files:**
- Modify: `modules/loop-agent/amplifier_module_loop_agent/agent_session.py` (if not done in 1.1)
- Test: `modules/loop-agent/tests/test_system_prompt_wiring.py`

**Context:** `build_environment_context()` produces a `<environment>` block with working dir, platform, git state, date, provider/model. This must be the second layer in the system prompt.

**Step 1: Write the failing test**

```python
@pytest.mark.asyncio
async def test_environment_context_in_system_prompt():
    """System prompt must contain <environment> block with working dir."""
    config = SessionConfig.from_dict({
        "system_prompt": "Base prompt.",
        "max_tool_rounds_per_input": 1,
    })
    provider = AsyncMock()
    provider.complete = AsyncMock(return_value=_make_text_response("done"))
    hooks = AsyncMock()
    hooks.emit = AsyncMock()

    session = AgentSession(
        config=config, provider=provider, tools={}, hooks=hooks,
        provider_name="anthropic", model="claude-sonnet-4-6",
    )
    await session.process_input("hello")

    request = provider.complete.call_args[0][0]
    system_content = request.messages[0].content
    assert "<environment>" in system_content
    assert "Working directory:" in system_content
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** (likely already done as part of Task 1.1 — verify the environment layer flows through)

**Step 4: Run — expect PASS**

**Step 5: Commit** (squash with 1.1 if done together)

---

### Task 1.3: Wire Project Doc Discovery into System Prompt

**Files:**
- Modify: `modules/loop-agent/amplifier_module_loop_agent/agent_session.py`
- Test: `modules/loop-agent/tests/test_system_prompt_wiring.py`

**Context:** `discover_project_docs()` walks from git root to CWD, loads AGENTS.md (always) plus provider-specific files (CLAUDE.md for Anthropic, .codex/instructions.md for OpenAI, GEMINI.md for Gemini). This is layer 4 of the 5-layer prompt.

**Step 1: Write the failing test**

```python
@pytest.mark.asyncio
async def test_project_docs_discovered_for_provider(tmp_path):
    """System prompt includes AGENTS.md content when present."""
    # Create a fake AGENTS.md in a temp dir
    agents_md = tmp_path / "AGENTS.md"
    agents_md.write_text("# Project Rules\nAlways use TDD.")

    config = SessionConfig.from_dict({
        "system_prompt": "Base.",
        "max_tool_rounds_per_input": 1,
        "working_dir": str(tmp_path),
    })
    provider = AsyncMock()
    provider.complete = AsyncMock(return_value=_make_text_response("done"))
    hooks = AsyncMock()
    hooks.emit = AsyncMock()

    session = AgentSession(
        config=config, provider=provider, tools={}, hooks=hooks,
        provider_name="anthropic",
    )
    await session.process_input("hello")

    request = provider.complete.call_args[0][0]
    system_content = request.messages[0].content
    assert "Always use TDD." in system_content
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** — In `_build_system_prompt()`, call `discover_project_docs(working_dir, provider_id)` and pass the result as the `project_docs` layer.

**Step 4: Run — expect PASS**

**Step 5: Commit**

```bash
git commit -m "feat(loop-agent): wire project doc discovery into system prompt"
```

---

### Task 1.4: Verify Hook-Based Tool Truncation Wiring

**Files:**
- Test: `modules/loop-agent/tests/test_truncation_wiring.py`

**Context:** `hooks-tool-truncation` registers on `tool:post` to truncate output. But `agent_session._execute_single_tool()` (line 277-324) calls `tool.execute()` and emits `AGENT_TOOL_CALL_END` — it does NOT emit a `tool:post` event that the truncation hook would intercept. We need to verify whether the hook system intercepts at the right level, or if the agent session needs to explicitly emit `tool:post` events.

**Step 1: Write the failing test**

```python
@pytest.mark.asyncio
async def test_tool_post_event_emitted_after_execution():
    """Agent session must emit tool:post event so truncation hook can intercept."""
    config = SessionConfig.from_dict({"max_tool_rounds_per_input": 1})
    provider = AsyncMock()
    provider.complete = AsyncMock(side_effect=[
        _make_tool_call_response("bash", {"command": "echo hi"}),
        _make_text_response("done"),
    ])
    hooks = AsyncMock()
    hooks.emit = AsyncMock()
    tool = MagicMock()
    tool.name = "bash"
    tool.execute = AsyncMock(return_value=ToolResult(success=True, output="hi"))

    session = AgentSession(
        config=config, provider=provider, tools={"bash": tool}, hooks=hooks,
    )
    await session.process_input("run it")

    # Check that tool:post was emitted (required for truncation hook)
    emit_calls = [call[0][0] for call in hooks.emit.call_args_list]
    assert "tool:post" in emit_calls, (
        f"tool:post not emitted. Events: {emit_calls}"
    )
```

**Step 2: Run — check if PASS or FAIL**

If FAIL: Add `tool:post` event emission in `_execute_single_tool()` after `tool.execute()` returns, passing the result so the truncation hook can modify it.

If PASS: The hook system is already intercepting at the right level. Document with a passing test and move on.

**Step 3: Implement if needed** — Add `await self._hooks.emit("tool:post", {"tool_name": tool_call.name, "result": result})` after tool execution.

**Step 4: Run — expect PASS**

**Step 5: Commit**

```bash
git commit -m "feat(loop-agent): emit tool:post events for truncation hook wiring"
```

---

### Task 1.5: Wire Fidelity Modes into CodergenBackend Adapter

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/backend.py`
- Test: `modules/loop-pipeline/tests/test_backend_fidelity.py`

**Context:** `backend.py` always creates a fresh session per node (line 92-93). The spec says fidelity mode controls whether to reuse a session (`full`), create a fresh one with a summary preamble (`compact`, `summary:*`), or create a minimal one (`truncate`). `fidelity.py` implements all the resolution and preamble-building logic but `backend.py` never calls it.

**Step 1: Write the failing test**

```python
@pytest.mark.asyncio
async def test_backend_uses_fidelity_preamble():
    """Backend builds a preamble from fidelity mode and prepends to prompt."""
    from amplifier_module_loop_pipeline.backend import AmplifierBackend
    from amplifier_module_loop_pipeline.graph import Node, Edge, Graph
    from amplifier_module_loop_pipeline.context import PipelineContext
    from amplifier_module_loop_pipeline.outcome import Outcome, StageStatus

    coordinator = MagicMock()
    spawn_fn = AsyncMock(return_value={"output": "done"})
    coordinator.get_capability = MagicMock(return_value=spawn_fn)

    backend = AmplifierBackend(coordinator, profiles={"anthropic": "profile-anthropic"})

    node = Node(id="impl", label="Implement", shape="box",
                attrs={"llm_provider": "anthropic", "fidelity": "compact"})
    context = PipelineContext()
    context.set("graph.goal", "Build feature X")

    # Give the backend completed node history so preamble has content
    backend._completed_nodes = {"plan": Outcome(status=StageStatus.SUCCESS, notes="Plan done")}

    result = await backend.run(node, "Implement the plan", context)

    # Verify spawn was called with a prompt that includes preamble
    spawn_call = spawn_fn.call_args
    instruction = spawn_call.kwargs.get("instruction", spawn_call[1].get("instruction", ""))
    assert "Goal:" in instruction  # compact preamble starts with Goal:
    assert "Implement the plan" in instruction
```

**Step 2: Run — expect FAIL** (backend doesn't call fidelity)

**Step 3: Implement**

In `backend.py`:
1. Import `resolve_fidelity`, `resolve_thread_key`, `build_preamble` from `.fidelity`.
2. Add `_completed_nodes: dict[str, Outcome]` and `_session_pool: dict[str, str]` fields to `__init__`.
3. In `run()`, before spawning:
   - Call `resolve_fidelity(node, incoming_edge, graph)` to get the mode.
   - If mode is `full`, check `_session_pool` for an existing session_id with the resolved thread key. If found, resume it.
   - Otherwise, call `build_preamble(fidelity, context, self._completed_nodes)` and prepend it to the prompt.
4. After spawn completes, record the session_id in `_session_pool` keyed by thread key.
5. Record the outcome in `_completed_nodes`.

Note: `run()` currently doesn't receive `incoming_edge` or `graph` — the `CodergenHandler` will need to pass these through. Update the `run()` signature to accept optional `edge` and `graph` parameters.

**Step 4: Run — expect PASS**

**Step 5: Commit**

```bash
git commit -m "feat(loop-pipeline): wire fidelity modes and session pooling into backend"
```

---

### Task 1.6: Wire Artifact Store into Pipeline Engine

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/engine.py`
- Test: `modules/loop-pipeline/tests/test_engine_artifacts.py`

**Context:** `engine.py` creates the `artifacts/` directory (line 442) but never instantiates `ArtifactStore`. Handlers and the backend should be able to store/retrieve artifacts through the engine.

**Step 1: Write the failing test**

```python
@pytest.mark.asyncio
async def test_engine_has_artifact_store(tmp_path):
    """Engine must expose an artifact store for handlers to use."""
    from amplifier_module_loop_pipeline.engine import PipelineEngine
    from amplifier_module_loop_pipeline.context import PipelineContext
    from amplifier_module_loop_pipeline.handlers import HandlerRegistry
    # ... (build a simple graph with start -> exit)

    engine = PipelineEngine(
        graph=graph, context=PipelineContext(),
        handler_registry=HandlerRegistry(),
        logs_root=str(tmp_path),
    )

    # Artifact store should exist and be usable
    assert hasattr(engine, 'artifact_store')
    assert engine.artifact_store is not None

    # Store something
    artifact = engine.artifact_store.store("test_output", "hello world")
    assert artifact.name == "test_output"
```

**Step 2: Run — expect FAIL**

**Step 3: Implement**

In `engine.py`:
1. Import `ArtifactStore` from `.artifacts`.
2. In `__init__`, create `self.artifact_store = ArtifactStore(logs_root)`.
3. Pass `artifact_store` to handlers that need it (via context or direct reference).

**Step 4: Run — expect PASS**

**Step 5: Commit**

```bash
git commit -m "feat(loop-pipeline): wire artifact store into pipeline engine"
```

---

### Task 1.7: Pass Hooks to Pipeline Engine

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/__init__.py`
- Test: `modules/loop-pipeline/tests/test_pipeline_hooks.py`

**Context:** `PipelineOrchestrator.execute()` (line 93-98) creates the `PipelineEngine` without passing `hooks`. The engine supports hooks (line 62, `hooks: Any | None = None`) but never receives them from the orchestrator.

**Step 1: Write the failing test**

```python
@pytest.mark.asyncio
async def test_pipeline_emits_events_via_hooks():
    """Pipeline orchestrator must pass hooks to engine for event emission."""
    orchestrator = PipelineOrchestrator(config={"dot_source": SIMPLE_DOT})
    hooks = AsyncMock()
    hooks.emit = AsyncMock()

    await orchestrator.execute(
        prompt="test", context=MagicMock(),
        providers={"mock": MagicMock()}, tools={}, hooks=hooks,
    )

    emit_events = [call[0][0] for call in hooks.emit.call_args_list]
    assert "pipeline:start" in emit_events
    assert "pipeline:complete" in emit_events
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** — Pass `hooks=hooks` to `PipelineEngine(...)` in `execute()`.

**Step 4: Run — expect PASS**

**Step 5: Commit**

```bash
git commit -m "feat(loop-pipeline): pass hooks to engine for pipeline event emission"
```

---

## Sprint 2: Implement the CRITICALs

**Goal:** Close the three gaps that block real-world usage. These require new code, not just wiring.

**Estimated effort:** ~6-8 hours total (3 independent tasks)

---

### Task 2.1: Streaming Support in loop-agent (GAP-AL-01)

**Files:**
- Modify: `modules/loop-agent/amplifier_module_loop_agent/agent_session.py`
- Modify: `modules/loop-agent/amplifier_module_loop_agent/config.py`
- Modify: `modules/loop-agent/amplifier_module_loop_agent/events.py`
- Create: `modules/loop-agent/tests/test_streaming.py`

**Context:** `agent_session.py` line 150 calls `provider.complete(request)` which blocks until the full response is ready. Users see nothing for 30-60s. The spec requires streaming via `Client.stream()` with `ASSISTANT_TEXT_DELTA` events for real-time feedback.

The Amplifier provider protocol already supports `stream=True` on `ChatRequest`, and `loop-streaming` shows the pattern for consuming streaming responses. The key insight: we don't need to change the `execute() -> str` return contract. Streaming is about **emitting events during execution**, not changing the return type.

**Step 1: Write the failing test**

```python
# tests/test_streaming.py
import pytest
from unittest.mock import AsyncMock, MagicMock

@pytest.mark.asyncio
async def test_streaming_emits_text_delta_events():
    """When streaming is enabled, agent emits agent:text_delta events."""
    config = SessionConfig.from_dict({
        "stream": True,
        "max_tool_rounds_per_input": 1,
    })

    # Mock a streaming provider response
    async def fake_complete(request, **kwargs):
        # Simulate that the provider returns a complete response
        # but we also check that stream=True was set on the request
        assert request.stream is True
        return _make_text_response("Hello world")

    provider = AsyncMock()
    provider.complete = AsyncMock(side_effect=fake_complete)
    hooks = AsyncMock()
    hooks.emit = AsyncMock()

    session = AgentSession(config=config, provider=provider, tools={}, hooks=hooks)
    result = await session.process_input("hi")

    # Should still return final text
    assert result == "Hello world"

    # Should have set stream=True on ChatRequest
    call_args = provider.complete.call_args
    request = call_args[0][0]
    assert request.stream is True
```

**Step 2: Run — expect FAIL** (stream not set on ChatRequest)

**Step 3: Implement**

1. Add `stream: bool = False` to `SessionConfig` (in `config.py`).
2. Add `AGENT_TEXT_DELTA = "agent:text_delta"` to `events.py`.
3. In `agent_session.py` `process_input()`, set `stream=self._config.stream` on the `ChatRequest`.
4. For the initial implementation: even with `stream=True`, use `provider.complete()` — the provider handles streaming internally and returns the accumulated response. The key addition is that the `ChatRequest.stream` flag is set, which tells providers to stream. Providers that support streaming will emit their own `content_block:delta` events via hooks.
5. Future enhancement: consume the stream iterator directly for providers that support `provider.stream()`, emitting `AGENT_TEXT_DELTA` events per chunk.

**Step 4: Run — expect PASS**

**Step 5: Commit**

```bash
git commit -m "feat(loop-agent): add streaming flag support to ChatRequest construction"
```

**Step 6: Write the advanced streaming test**

```python
@pytest.mark.asyncio
async def test_streaming_text_deltas_emitted_from_hook_events():
    """When using streaming orchestrators, text delta events flow through hooks."""
    # This test verifies the event flow, not the provider internals
    config = SessionConfig.from_dict({"stream": True, "max_tool_rounds_per_input": 1})
    provider = AsyncMock()
    provider.complete = AsyncMock(return_value=_make_text_response("streamed result"))
    hooks = AsyncMock()
    hooks.emit = AsyncMock()

    session = AgentSession(config=config, provider=provider, tools={}, hooks=hooks)
    result = await session.process_input("test")

    assert result == "streamed result"
    # The agent:assistant_text_end event should still fire with the final text
    emit_events = [call[0][0] for call in hooks.emit.call_args_list]
    assert "agent:assistant_text_end" in emit_events
```

**Step 7: Run — expect PASS**

**Step 8: Commit**

```bash
git commit -m "feat(loop-agent): streaming event flow with text delta support"
```

---

### Task 2.2: Provider-Aligned Tool Presentation (GAP-AL-03)

**Files:**
- Modify: `modules/loop-agent/amplifier_module_loop_agent/__init__.py`
- Modify: `modules/loop-agent/amplifier_module_loop_agent/agent_session.py`
- Modify: `modules/loop-agent/amplifier_module_loop_agent/config.py`
- Create: `modules/loop-agent/tests/test_provider_aligned_tools.py`

**Context:** The profile bundles (in `profiles/`) define which tools each provider gets, and the system prompts (in `context/`) describe how each provider should use its tools. But `AgentOrchestrator.execute()` passes ALL mounted tools to the session (line 98-104). The orchestrator doesn't read the profile's system prompt or filter tools.

The "sessions all the way down" architecture means **bundle composition handles this** — when the pipeline spawns a child session with `profile=attractor-profile-anthropic`, that profile's bundle YAML mounts only Anthropic-appropriate tools. The loop-agent orchestrator doesn't need to filter. It just needs to:
1. Read the system prompt from profile context (the `context/*.md` files).
2. Present whatever tools were mounted by the bundle.

**Step 1: Write the failing test**

```python
# tests/test_provider_aligned_tools.py
@pytest.mark.asyncio
async def test_system_prompt_from_config():
    """Agent reads base system prompt from orchestrator config."""
    config_dict = {
        "system_prompt": "You are a Claude Code agent. Use edit_file for edits.",
        "max_tool_rounds_per_input": 1,
    }
    provider = AsyncMock()
    provider.complete = AsyncMock(return_value=_make_text_response("ok"))
    hooks = AsyncMock()
    hooks.emit = AsyncMock()

    orchestrator = AgentOrchestrator(MagicMock(), config_dict)
    result = await orchestrator.execute(
        prompt="hello",
        context=MagicMock(),
        providers={"anthropic": provider},
        tools={},
        hooks=hooks,
    )

    request = provider.complete.call_args[0][0]
    system_msg = request.messages[0]
    assert system_msg.role == "system"
    assert "Claude Code agent" in system_msg.content
```

**Step 2: Run — expect FAIL**

**Step 3: Implement**

1. In `config.py`, add `system_prompt: str = ""` and `working_dir: str = ""` to `SessionConfig.from_dict()`.
2. In `__init__.py` `AgentOrchestrator.execute()`, extract provider name from providers dict key and pass it to `AgentSession.__init__()`.
3. In `agent_session.py`:
   - Accept `provider_name` and `model` in `__init__`.
   - In `_convert_history_to_messages()` (or a new `_build_messages()` method), prepend a system `Message` built from `_build_system_prompt()`.
   - `_build_system_prompt()` calls `build_system_prompt(base_prompt=config.system_prompt, environment=build_environment_context(...), ...)`.

**Step 4: Run — expect PASS**

**Step 5: Write tool presentation test**

```python
@pytest.mark.asyncio
async def test_only_mounted_tools_in_request():
    """Only tools mounted by the bundle appear in the ChatRequest."""
    provider = AsyncMock()
    provider.complete = AsyncMock(return_value=_make_text_response("ok"))
    hooks = AsyncMock()
    hooks.emit = AsyncMock()

    # Simulate an OpenAI profile: only apply_patch and bash are mounted
    tools = {
        "apply_patch": _make_tool("apply_patch"),
        "bash": _make_tool("bash"),
    }

    orchestrator = AgentOrchestrator(MagicMock(), {"max_tool_rounds_per_input": 1})
    await orchestrator.execute("hello", MagicMock(), {"openai": provider}, tools, hooks)

    request = provider.complete.call_args[0][0]
    tool_names = [t.name for t in request.tools]
    assert "apply_patch" in tool_names
    assert "bash" in tool_names
    assert len(tool_names) == 2  # Only what was mounted
```

**Step 6: Run — expect PASS** (this should already work since the orchestrator uses whatever tools are passed)

**Step 7: Commit**

```bash
git commit -m "feat(loop-agent): provider-aligned tool presentation via profile config"
```

---

### Task 2.3: Manager Loop Handler Full Implementation (GAP-PL-01)

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/handlers/manager_loop.py`
- Create: `modules/loop-pipeline/tests/test_manager_loop.py`

**Context:** The manager loop (shape=house) is the core Attractor supervisor pattern. It spawns a child pipeline, observes telemetry, evaluates a guard function, and steers the child. Currently it's a stub returning SUCCESS.

The implementation uses the "sessions all the way down" pattern — the manager spawns the child pipeline as a sub-session, then enters an observation loop.

**Step 1: Write the failing test**

```python
# tests/test_manager_loop.py
@pytest.mark.asyncio
async def test_manager_loop_runs_child_pipeline():
    """Manager loop spawns and runs a child pipeline to completion."""
    node = Node(
        id="manager", label="Sprint Manager", shape="house",
        attrs={
            "stack.child_dotfile": "child.dot",
            "manager.max_cycles": "3",
            "manager.poll_interval": "1s",
            "manager.stop_condition": "outcome = success",
        },
    )

    # Mock the backend to simulate a successful child pipeline
    mock_backend = AsyncMock()
    mock_backend.run_pipeline = AsyncMock(
        return_value=Outcome(status=StageStatus.SUCCESS, notes="Child completed")
    )

    handler = ManagerLoopHandler(backend=mock_backend)
    context = PipelineContext()
    context.set("graph.goal", "Build feature X")

    result = await handler.execute(node, context, graph, "/tmp/logs")

    assert result.status == StageStatus.SUCCESS
    mock_backend.run_pipeline.assert_called_once()
```

**Step 2: Run — expect FAIL** (stub returns SUCCESS without running anything)

**Step 3: Implement**

Replace the stub with the full manager loop:

```python
class ManagerLoopHandler:
    def __init__(self, backend: Any | None = None) -> None:
        self._backend = backend

    async def execute(
        self, node: Node, context: PipelineContext,
        graph: Graph, logs_root: str,
    ) -> Outcome:
        max_cycles = int(node.attrs.get("manager.max_cycles", 10))
        poll_interval_s = _parse_duration(node.attrs.get("manager.poll_interval", "45s"))
        stop_condition = node.attrs.get("manager.stop_condition", "")
        child_dotfile = node.attrs.get("stack.child_dotfile", "")

        if not child_dotfile or self._backend is None:
            return Outcome(
                status=StageStatus.FAIL,
                failure_reason="Manager loop requires stack.child_dotfile and a backend",
            )

        # Read child DOT source
        dot_path = os.path.join(os.path.dirname(logs_root), child_dotfile)
        if not os.path.exists(dot_path):
            return Outcome(
                status=StageStatus.FAIL,
                failure_reason=f"Child dotfile not found: {dot_path}",
            )

        with open(dot_path) as f:
            child_dot = f.read()

        # Run cycles
        last_outcome = None
        for cycle in range(1, max_cycles + 1):
            logger.info("Manager '%s' cycle %d/%d", node.id, cycle, max_cycles)

            # Run child pipeline
            last_outcome = await self._backend.run_pipeline(
                child_dot, context, logs_root=os.path.join(logs_root, node.id, f"cycle_{cycle}"),
            )

            # Check stop condition
            if _evaluate_stop_condition(stop_condition, last_outcome, context):
                return Outcome(
                    status=last_outcome.status,
                    notes=f"Manager completed in {cycle} cycles",
                    context_updates={"last_stage": node.id, "manager.cycles": cycle},
                )

            # Wait before next cycle (unless this was the last)
            if cycle < max_cycles:
                await asyncio.sleep(poll_interval_s)

        # Max cycles exhausted
        return Outcome(
            status=StageStatus.PARTIAL_SUCCESS if last_outcome and last_outcome.is_success else StageStatus.FAIL,
            failure_reason=f"Manager exhausted {max_cycles} cycles",
            notes=f"Last cycle outcome: {last_outcome.status.value if last_outcome else 'none'}",
            context_updates={"last_stage": node.id, "manager.cycles": max_cycles},
        )
```

**Step 4: Run — expect PASS**

**Step 5: Write additional tests**

```python
@pytest.mark.asyncio
async def test_manager_loop_retries_on_failure():
    """Manager retries when child fails and stop condition not met."""
    # ... test that cycles > 1 when first cycle fails

@pytest.mark.asyncio
async def test_manager_loop_stops_on_condition():
    """Manager stops early when stop_condition is satisfied."""
    # ... test that stop_condition evaluation works

@pytest.mark.asyncio
async def test_manager_loop_exhausts_max_cycles():
    """Manager returns PARTIAL_SUCCESS/FAIL when max cycles exhausted."""
    # ... test max_cycles boundary
```

**Step 6: Run — expect PASS**

**Step 7: Commit**

```bash
git commit -m "feat(loop-pipeline): implement manager loop handler with cycle-based supervision"
```

---

## Sprint 3: HIGH Gaps

**Goal:** Close the HIGH-severity gaps. Depends on Sprint 1 and Sprint 2 being complete.

**Estimated effort:** ~4 hours total

---

### Task 3.1: Subagent Lifecycle Tools (GAP-AL-02)

**Files:**
- Create: `modules/loop-agent/amplifier_module_loop_agent/subagent_tools.py`
- Test: `modules/loop-agent/tests/test_subagent_tools.py`

**Context:** The spec requires `spawn_agent`, `send_input`, `wait`, `close_agent` as separate tools for interactive subagent management. Currently, tool-delegate only supports spawn-and-block (fire and wait for completion). The interactive lifecycle allows the host to spawn an agent, send multiple inputs, and close it.

**Implementation approach:** Build these as internal tools that the loop-agent orchestrator registers when configured. They use the existing `session.spawn` and `session.resume` capabilities from Amplifier foundation.

**Step 1: Write the failing test**

```python
@pytest.mark.asyncio
async def test_spawn_agent_returns_session_id():
    """spawn_agent creates a session and returns its ID without blocking."""
    tool = SpawnAgentTool(coordinator=mock_coordinator)
    result = await tool.execute({
        "agent": "coding-agent",
        "instruction": "Plan the feature",
    })
    assert result.success
    assert "session_id" in result.output
```

**Step 2-7:** Implement `SpawnAgentTool`, `SendInputTool`, `WaitTool`, `CloseAgentTool` using the existing spawn/resume capability pattern from tool-delegate. Each tool is thin — it delegates to the session management capability.

**Commit message:** `feat(loop-agent): add interactive subagent lifecycle tools`

---

### Task 3.2: Remaining Parallel Handler Join/Error Policies (GAP-PL-07)

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/handlers/parallel.py`
- Test: `modules/loop-pipeline/tests/test_parallel_policies.py`

**Context:** Only `wait_all` and `first_success` join policies are implemented. Missing: `k_of_n` (succeed when K of N branches succeed), `quorum` (majority). Only `continue` error policy (catch and continue). Missing: `fail_fast` (cancel remaining on first failure), `ignore` (treat failures as successes).

**Step 1: Write failing tests for each policy**

```python
@pytest.mark.asyncio
async def test_k_of_n_policy_succeeds_when_threshold_met():
    """k_of_n succeeds when k branches out of n succeed."""
    # Node with join_policy="k_of_n", join_k=2
    # 3 branches: 2 succeed, 1 fails → SUCCESS

@pytest.mark.asyncio
async def test_fail_fast_cancels_remaining():
    """fail_fast error policy cancels remaining branches on first failure."""
    # 3 branches: first fails → remaining should be cancelled
```

**Step 2-5:** Add `k_of_n`, `quorum`, `fail_fast`, `ignore` to `_apply_join_policy()`. For `fail_fast`, use `asyncio.FIRST_EXCEPTION` gather pattern and cancel remaining tasks.

**Commit message:** `feat(loop-pipeline): add k_of_n, quorum, fail_fast, ignore policies to parallel handler`

---

## Sprint 4: Integration Testing

**Goal:** Run the full stack end-to-end with real providers and in a shadow environment.

**Estimated effort:** ~3 hours

---

### Task 4.1: Real API Integration Test (Simple Pipeline)

**Files:**
- Create: `modules/loop-pipeline/tests/integration/test_simple_pipeline_e2e.py`

**Context:** Run a minimal DOT pipeline end-to-end with a real provider (Anthropic or OpenAI). This tests the full stack: DOT parsing → graph validation → node execution → backend spawns coding agent → agent calls real LLM → agent executes tools → outcome parsing → edge selection → pipeline completes.

```python
@pytest.mark.integration
@pytest.mark.asyncio
async def test_simple_two_node_pipeline_with_real_provider():
    """End-to-end: plan → implement pipeline with real Anthropic API."""
    dot = '''
    digraph simple {
        start [shape=Mdiamond]
        plan [prompt="List 3 steps to create a hello world Python script"]
        finish [shape=Msquare]
        start -> plan
        plan -> finish
    }
    '''
    # ... set up real provider, run pipeline, assert SUCCESS outcome
```

### Task 4.2: Shadow Environment Validation

**Files:**
- Create: `tests/integration/test_shadow_validation.sh`

**Context:** Create a shadow environment with all 8+ repos (core, providers, tools, loop-agent, loop-pipeline), install the bundle, and run the full test suite. Validates that all cross-repo dependencies resolve correctly.

Steps:
1. Create shadow environment with `amplifier-bundle-attractor` as local source
2. Install all modules from the bundle's `modules/` directory
3. Run `uv run pytest` in each module directory
4. Verify all imports resolve correctly
5. Run a simple pipeline end-to-end inside the shadow

### Task 4.3: Cross-Module Integration Test

**Files:**
- Create: `tests/integration/test_loop_agent_with_tools.py`

**Context:** Test that loop-agent correctly works with real Amplifier tools (tool-filesystem, tool-bash, tool-search) mounted via the bundle system.

```python
@pytest.mark.integration
@pytest.mark.asyncio
async def test_agent_uses_filesystem_tool():
    """Agent can read a file using tool-filesystem through the loop-agent orchestrator."""
    # Set up with mock provider that requests read_file tool call
    # Verify the tool executes and the result flows back to the agent
```

**Commit message:** `test: add integration tests for full pipeline and cross-module scenarios`

---

## Sprint 5: MEDIUM Gaps (Ongoing)

**Goal:** Address remaining MEDIUM gaps incrementally. These can be done after the system is working end-to-end.

**Estimated effort:** ~1-2 hours per task, done over time

---

### Task 5.1: apply_patch Fuzzy Matching (GAP-AL-06)

**Files:**
- Modify: `modules/tool-apply-patch/amplifier_module_tool_apply_patch/parser.py`

**Context:** The v4a patch parser currently requires exact hunk matching (the context lines in the patch must match the file exactly). Fuzzy matching would find the closest match when lines have been slightly modified (e.g., whitespace changes).

**Implementation:** Add a `_fuzzy_find_hunk()` method that uses difflib.SequenceMatcher to find the best match location when exact matching fails. Only activate when exact match fails — exact is always preferred.

---

### Task 5.2: Checkpoint Crash Recovery Testing (GAP-PL-02)

**Files:**
- Create: `modules/loop-pipeline/tests/test_checkpoint_recovery.py`

**Context:** `engine.py` has checkpoint save/load logic (lines 363-432) but it's never been tested with a simulated crash and resume. Write tests that:
1. Run a pipeline partway through
2. Kill the engine (simulate crash)
3. Create a new engine instance
4. Resume from the checkpoint
5. Verify it skips completed nodes and continues

---

### Task 5.3: Variable Expansion Beyond $goal (GAP-PL-03)

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/transforms.py`

**Context:** Currently only `$goal` is expanded. Add support for `$context.*` variables (e.g., `$context.last_response`) by resolving against the PipelineContext.

---

### Task 5.4: Fan-In LLM-Based Ranking (GAP-PL-04)

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/handlers/fan_in.py`

**Context:** Currently uses heuristic ranking (status-based). The spec also supports LLM-based ranking where the LLM evaluates which candidate is best. Add a `ranking_mode` attribute (`heuristic` | `llm`) and implement the LLM path using the backend.

---

### Task 5.5: Human Gate with Real Approval System (GAP-PL-05)

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/handlers/human.py`

**Context:** The human gate handler uses an `Interviewer` protocol. Wire it to Amplifier's `ask_user` hook action so it integrates with the CLI approval flow.

---

### Task 5.6: Pipeline Event Data Field Completeness (GAP-PL-10)

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/engine.py`

**Context:** Audit all `_emit()` calls against the spec's event data requirements. Add missing fields like `node_type`, `prompt_length`, `response_length`, `retry_count` to the appropriate events.

---

## Summary: What to Build in What Order

| Sprint | Tasks | Effort | Priority |
|--------|-------|--------|----------|
| **1: Wire the Unwired** | 7 tasks (1.1-1.7) | ~2h | **DO FIRST** — fastest wins |
| **2a: Streaming** | 1 task (2.1) | ~2h | **CRITICAL** — can parallel with 2b, 2c |
| **2b: Provider-Aligned Tools** | 1 task (2.2) | ~2h | **CRITICAL** — can parallel with 2a, 2c |
| **2c: Manager Loop** | 1 task (2.3) | ~3h | **CRITICAL** — can parallel with 2a, 2b |
| **3: HIGH gaps** | 2 tasks (3.1-3.2) | ~4h | Needs Sprint 1+2 |
| **4: Integration Testing** | 3 tasks (4.1-4.3) | ~3h | Needs Sprint 3 |
| **5: MEDIUM gaps** | 6 tasks (5.1-5.6) | ~8h | Ongoing after Sprint 4 |

**Total for "working end-to-end":** Sprints 1-4 = ~14 hours of agent work
**Total including all MEDIUM gaps:** ~22 hours

---

## Execution Approach

**Recommended: Subagent-driven parallel execution**

1. **Dispatch Sprint 1 tasks** (all 7 are independent within the sprint) via implementer agents
2. **Dispatch Sprint 2a, 2b, 2c** in parallel (they're independent)
3. **After Sprint 1+2 merge**, dispatch Sprint 3 tasks
4. **After Sprint 3**, run Sprint 4 integration testing
5. **Sprint 5** is ongoing — pick up tasks as time allows

**Review cadence:**
- After each sprint completes, run `spec-reviewer` to verify against the nlspec
- After each sprint, run `code-quality-reviewer` for quality assessment
- After Sprint 4, run shadow environment validation via `shadow-operator` + `amplifier-smoke-test`
