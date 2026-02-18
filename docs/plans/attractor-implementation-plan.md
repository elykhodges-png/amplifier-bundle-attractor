# Attractor on Amplifier: Implementation Plan

> **Execution:** Use the subagent-driven-development workflow to implement this plan.

**Goal:** Implement the full Attractor pipeline engine and coding agent loop as native Amplifier modules, following the "sessions all the way down" architecture where the pipeline itself is an agent that spawns coding agent sub-sessions.

**Architecture:**
```
Any Amplifier Session (CLI, automation, another agent)
  └── delegate(agent="attractor-pipeline", instruction="run spec.dot")
        └── Pipeline Session (loop-pipeline orchestrator + LLM for edge classification)
              ├── spawn(profile=anthropic) → Coding Agent (loop-agent + Anthropic tools)
              ├── spawn(profile=openai) → Coding Agent (loop-agent + OpenAI tools)
              └── ... per node in the DOT graph
```

**Tech Stack:** Python 3.11+, Amplifier module system (Pydantic models, async/await, Protocol classes), existing Amplifier tools (filesystem, bash, search, delegate)

**Specs:** `attractor/coding-agent-loop-spec.md` (155 requirements), `attractor/attractor-spec.md` (281 requirements)

**Completed Prerequisites:**
- Phase 1 (kernel vocabulary): `amplifier-core` PR #10 merged — Usage fields, ChatRequest fields, error taxonomy
- Phase 2 (provider improvements): 9 PRs merged — error translation, retry, usage extraction, reasoning_effort across all providers and orchestrators

---

## Critical Path & Dependency Graph

```
Phase 1 (loop-agent core) ──────────────────────────┐
Phase 2 (tool extensions) ──────┐                    │
Phase 3 (provider profiles) ────┤                    │
                                ├─→ Phase 4 (steering + detection + truncation)
Phase 5 (loop-pipeline core) ───┤          │
                                │          ▼
                                ├─→ Phase 6 (pipeline features)
                                │          │
                                │          ▼
                                └─→ Phase 7 (integration + parity)
```

**Parallelizable:** Phases 1, 2, 3, and 5 are independent and can run concurrently.
**Sequential:** Phase 4 requires Phases 1+2+3. Phase 6 requires Phases 4+5. Phase 7 requires Phase 6.

---

## Phase 1: Core loop-agent Orchestrator + State Machine

**Goal:** A working coding agent orchestrator with session state machine, core agentic loop, and basic tool execution — enough to run a single coding task to completion.

**New repo:** `amplifier-module-loop-agent/`

**Spec coverage:** ARCH-001–010, SESS-001–017, CFG-001–009, TURN-001–005, LOOP-001–023, STOP-001–005, REASON-001–007, EVENT-001–009, ERR-001–013, SHUT-001–009

---

### Task 1.1: Module Skeleton + Mount Function

**Files:**
- Create: `amplifier-module-loop-agent/amplifier_module_loop_agent/__init__.py`
- Create: `amplifier-module-loop-agent/pyproject.toml`
- Create: `amplifier-module-loop-agent/tests/__init__.py`
- Test: `amplifier-module-loop-agent/tests/test_mount.py`

**Reference:** `amplifier-module-loop-basic/amplifier_module_loop_basic/__init__.py` lines 1–87 (mount pattern)

**Step 1: Write failing test**

```python
# tests/test_mount.py
import pytest
from unittest.mock import AsyncMock, MagicMock

@pytest.mark.asyncio
async def test_mount_registers_orchestrator():
    coordinator = MagicMock()
    coordinator.mount = MagicMock()
    from amplifier_module_loop_agent import mount
    await mount(coordinator, config={})
    coordinator.mount.assert_called_once()
    args = coordinator.mount.call_args
    assert args[0][0] == "orchestrator"
```

**Step 2: Run test — expect FAIL** (`ModuleNotFoundError`)

```bash
cd amplifier-module-loop-agent && uv run pytest tests/test_mount.py -v
```

**Step 3: Write minimal implementation**

`pyproject.toml`:
```toml
[project]
name = "amplifier-module-loop-agent"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["amplifier-core>=1.0.0"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

`__init__.py`:
```python
"""Attractor coding agent loop orchestrator.

A task-oriented agentic loop with session state machine, steering,
loop detection, and provider-aligned tool profiles.

Implements the coding-agent-loop-spec from the Attractor nlspec.
"""

from __future__ import annotations

import logging
from typing import Any

logger = logging.getLogger(__name__)


async def mount(coordinator: Any, config: dict[str, Any] | None = None) -> None:
    """Mount the loop-agent orchestrator."""
    cfg = config or {}
    orchestrator = AgentOrchestrator(coordinator, cfg)
    coordinator.mount("orchestrator", orchestrator)
    logger.info("loop-agent orchestrator mounted")
```

**Step 4: Run test — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: loop-agent module skeleton with mount function"
```

---

### Task 1.2: Session State Machine

**Files:**
- Modify: `amplifier_module_loop_agent/__init__.py`
- Test: `tests/test_session_state.py`

**Spec coverage:** SESS-007–015 (SessionState enum and transitions)

**Step 1: Write failing test**

```python
# tests/test_session_state.py
import pytest
from amplifier_module_loop_agent.state import SessionState, SessionStateMachine

def test_initial_state_is_idle():
    sm = SessionStateMachine()
    assert sm.state == SessionState.IDLE

def test_submit_transitions_idle_to_processing():
    sm = SessionStateMachine()
    sm.submit()
    assert sm.state == SessionState.PROCESSING

def test_complete_transitions_processing_to_idle():
    sm = SessionStateMachine()
    sm.submit()
    sm.complete()
    assert sm.state == SessionState.IDLE

def test_error_transitions_processing_to_closed():
    sm = SessionStateMachine()
    sm.submit()
    sm.fatal_error()
    assert sm.state == SessionState.CLOSED

def test_abort_from_any_state_goes_to_closed():
    sm = SessionStateMachine()
    sm.submit()
    sm.abort()
    assert sm.state == SessionState.CLOSED

def test_awaiting_input_transition():
    sm = SessionStateMachine()
    sm.submit()
    sm.await_input()
    assert sm.state == SessionState.AWAITING_INPUT

def test_resume_from_awaiting():
    sm = SessionStateMachine()
    sm.submit()
    sm.await_input()
    sm.resume_input()
    assert sm.state == SessionState.PROCESSING

def test_invalid_transition_raises():
    sm = SessionStateMachine()
    with pytest.raises(ValueError, match="Invalid transition"):
        sm.complete()  # Can't complete from IDLE

def test_close_from_idle():
    sm = SessionStateMachine()
    sm.close()
    assert sm.state == SessionState.CLOSED
```

**Step 2: Run — expect FAIL** (`ModuleNotFoundError` for `state`)

**Step 3: Write implementation**

Create `amplifier_module_loop_agent/state.py`:

```python
"""Session state machine for the coding agent loop.

Spec: SESS-007 through SESS-015.
"""
from __future__ import annotations
from enum import Enum


class SessionState(Enum):
    IDLE = "idle"
    PROCESSING = "processing"
    AWAITING_INPUT = "awaiting_input"
    CLOSED = "closed"


# Valid transitions: {from_state: {trigger: to_state}}
_TRANSITIONS: dict[SessionState, dict[str, SessionState]] = {
    SessionState.IDLE: {
        "submit": SessionState.PROCESSING,
        "close": SessionState.CLOSED,
        "abort": SessionState.CLOSED,
    },
    SessionState.PROCESSING: {
        "complete": SessionState.IDLE,
        "await_input": SessionState.AWAITING_INPUT,
        "fatal_error": SessionState.CLOSED,
        "abort": SessionState.CLOSED,
    },
    SessionState.AWAITING_INPUT: {
        "resume_input": SessionState.PROCESSING,
        "abort": SessionState.CLOSED,
    },
    SessionState.CLOSED: {},
}


class SessionStateMachine:
    """Enforces the session state transition rules."""

    def __init__(self) -> None:
        self._state = SessionState.IDLE

    @property
    def state(self) -> SessionState:
        return self._state

    def _transition(self, trigger: str) -> None:
        transitions = _TRANSITIONS.get(self._state, {})
        next_state = transitions.get(trigger)
        if next_state is None:
            raise ValueError(
                f"Invalid transition: {trigger!r} from {self._state.value}"
            )
        self._state = next_state

    def submit(self) -> None:
        self._transition("submit")

    def complete(self) -> None:
        self._transition("complete")

    def await_input(self) -> None:
        self._transition("await_input")

    def resume_input(self) -> None:
        self._transition("resume_input")

    def fatal_error(self) -> None:
        self._transition("fatal_error")

    def abort(self) -> None:
        self._transition("abort")

    def close(self) -> None:
        self._transition("close")
```

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: session state machine with transition enforcement"
```

---

### Task 1.3: Turn Types and Session History

**Files:**
- Create: `amplifier_module_loop_agent/turns.py`
- Test: `tests/test_turns.py`

**Spec coverage:** TURN-001–005, SESS-003–005

**Step 1: Write failing test**

```python
# tests/test_turns.py
from amplifier_module_loop_agent.turns import (
    UserTurn, AssistantTurn, ToolResultsTurn, SystemTurn, SteeringTurn, SessionHistory,
)

def test_user_turn_has_required_fields():
    t = UserTurn(content="hello")
    assert t.content == "hello"
    assert t.timestamp is not None

def test_assistant_turn_stores_tool_calls():
    t = AssistantTurn(content="I'll help", tool_calls=[{"id": "1", "name": "read_file"}])
    assert len(t.tool_calls) == 1

def test_session_history_append_and_count():
    h = SessionHistory()
    h.append(UserTurn(content="hello"))
    h.append(AssistantTurn(content="hi"))
    assert len(h) == 2
    assert h.turn_count == 2

def test_steering_turn():
    t = SteeringTurn(content="try a different approach")
    assert t.content == "try a different approach"
```

**Step 2: Run — expect FAIL**

**Step 3: Implement `turns.py`** with dataclasses for each turn type and `SessionHistory` as an ordered list with steering/follow-up queues.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: turn types and session history"
```

---

### Task 1.4: Session Configuration

**Files:**
- Create: `amplifier_module_loop_agent/config.py`
- Test: `tests/test_config.py`

**Spec coverage:** CFG-001–009

**Step 1: Write failing test**

```python
# tests/test_config.py
from amplifier_module_loop_agent.config import AgentConfig

def test_defaults():
    c = AgentConfig()
    assert c.max_turns == 0  # unlimited
    assert c.max_tool_rounds_per_input == 200
    assert c.default_command_timeout_ms == 10_000
    assert c.max_command_timeout_ms == 600_000
    assert c.reasoning_effort is None
    assert c.enable_loop_detection is True
    assert c.loop_detection_window == 10
    assert c.max_subagent_depth == 1

def test_from_dict():
    c = AgentConfig.from_dict({"max_turns": 50, "reasoning_effort": "high"})
    assert c.max_turns == 50
    assert c.reasoning_effort == "high"

def test_tool_output_limits_override():
    c = AgentConfig.from_dict({"tool_output_limits": {"shell": 50000}})
    assert c.get_tool_output_limit("shell") == 50000
    assert c.get_tool_output_limit("read_file") == 50000  # default
```

**Step 2: Run — expect FAIL**

**Step 3: Implement `config.py`** with a Pydantic model or dataclass, including default tool output limits from spec (TRUNC-005): read_file=50000, shell=30000, grep=20000, glob=20000, edit_file=10000, apply_patch=10000, write_file=1000, spawn_agent=20000.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: agent session configuration with spec defaults"
```

---

### Task 1.5: Core Agentic Loop (Main Execute Method)

**Files:**
- Modify: `amplifier_module_loop_agent/__init__.py`
- Test: `tests/test_execute.py`

**Spec coverage:** LOOP-001–021, STOP-001–005, ARCH-007–008

This is the heart of the orchestrator. It implements the `Orchestrator` protocol's `execute()` method.

**Reference:** `amplifier-module-loop-basic/amplifier_module_loop_basic/__init__.py` lines 88–750 (the existing basic loop)

**Step 1: Write failing test**

```python
# tests/test_execute.py
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from amplifier_core.message_models import ChatResponse, Message, Usage
from amplifier_core.models import ToolResult

@pytest.mark.asyncio
async def test_natural_completion_no_tools():
    """Agent returns text without tool calls → loop exits."""
    orchestrator, context, providers, tools, hooks = make_test_harness(
        responses=[
            ChatResponse(
                content=[{"type": "text", "text": "Done!"}],
                tool_calls=[],
                usage=Usage(input_tokens=10, output_tokens=5, total_tokens=15),
            )
        ]
    )
    result = await orchestrator.execute("Write hello.py", context, providers, tools, hooks)
    assert result == "Done!"

@pytest.mark.asyncio
async def test_tool_call_then_completion():
    """Agent calls a tool, gets result, then completes."""
    orchestrator, context, providers, tools, hooks = make_test_harness(
        responses=[
            ChatResponse(
                content=[],
                tool_calls=[{"id": "tc1", "name": "write_file", "arguments": {"file_path": "hello.py", "content": "print('hi')"}}],
                usage=Usage(input_tokens=10, output_tokens=5, total_tokens=15),
            ),
            ChatResponse(
                content=[{"type": "text", "text": "Created hello.py"}],
                tool_calls=[],
                usage=Usage(input_tokens=20, output_tokens=10, total_tokens=30),
            ),
        ]
    )
    result = await orchestrator.execute("Write hello.py", context, providers, tools, hooks)
    assert "hello.py" in result

@pytest.mark.asyncio
async def test_max_iterations_stops_loop():
    """Loop stops after max_tool_rounds_per_input."""
    orchestrator, context, providers, tools, hooks = make_test_harness(
        config={"max_tool_rounds_per_input": 2},
        responses=[
            # Two tool-call rounds, then we expect the loop to stop
            ChatResponse(content=[], tool_calls=[{"id": "tc1", "name": "read_file", "arguments": {}}],
                        usage=Usage(input_tokens=10, output_tokens=5, total_tokens=15)),
            ChatResponse(content=[], tool_calls=[{"id": "tc2", "name": "read_file", "arguments": {}}],
                        usage=Usage(input_tokens=10, output_tokens=5, total_tokens=15)),
            ChatResponse(content=[{"type": "text", "text": "Final answer"}], tool_calls=[],
                        usage=Usage(input_tokens=10, output_tokens=5, total_tokens=15)),
        ]
    )
    result = await orchestrator.execute("do something", context, providers, tools, hooks)
    assert result is not None
```

**Step 2: Run — expect FAIL**

**Step 3: Implement the `AgentOrchestrator.execute()` method**

The core loop structure (pseudocode from spec):
```
execute(prompt, context, providers, tools, hooks):
    state_machine.submit()
    append UserTurn(prompt)
    emit USER_INPUT
    drain_steering()
    round_count = 0

    while True:
        if round_count >= config.max_tool_rounds_per_input:
            emit TURN_LIMIT; break
        if config.max_turns > 0 and history.turn_count >= config.max_turns:
            emit TURN_LIMIT; break
        if abort_signaled:
            break

        # Build request
        messages = convert_history_to_messages()
        chat_request = ChatRequest(
            messages=messages, tools=tool_specs,
            model=config.model, reasoning_effort=config.reasoning_effort,
        )

        # Call LLM (single-shot, no SDK tool loop)
        response = await provider.complete(chat_request)

        # Record assistant turn
        append AssistantTurn(response.content, response.tool_calls, ...)
        emit ASSISTANT_TEXT_END

        # Natural completion check
        tool_calls = provider.parse_tool_calls(response)
        if not tool_calls:
            break

        # Execute tools
        results = await execute_tool_calls(tool_calls, tools, hooks)
        append ToolResultsTurn(results)
        round_count += 1

        # Drain steering
        drain_steering()

    # Follow-up processing
    if followup_queue:
        return await process_followup()

    state_machine.complete()
    emit SESSION_END
    return final_text
```

**Key differences from loop-basic:**
- Uses `SessionStateMachine` for state tracking
- Has `steering_queue` and `followup_queue`
- Tracks `round_count` per input (not per session)
- Records typed Turn objects in history
- Will add loop detection in Phase 4

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: core agentic loop with state machine and tool execution"
```

---

### Task 1.6: Event Emission

**Files:**
- Create: `amplifier_module_loop_agent/events.py`
- Modify: `amplifier_module_loop_agent/__init__.py` (wire events into loop)
- Test: `tests/test_events.py`

**Spec coverage:** EVENT-001–009

**Step 1: Write failing test**

```python
# tests/test_events.py
import pytest

@pytest.mark.asyncio
async def test_session_lifecycle_events():
    """SESSION_START and SESSION_END bracket execution."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        responses=[text_response("done")]
    )
    await orchestrator.execute("hello", ctx, providers, tools, hooks)
    events = hooks.emitted_events
    event_names = [e[0] for e in events]
    assert "agent:session_start" in event_names
    assert "agent:session_end" in event_names

@pytest.mark.asyncio
async def test_tool_call_events():
    """TOOL_CALL_START and TOOL_CALL_END bracket tool execution."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        responses=[tool_response("read_file", {}), text_response("done")]
    )
    await orchestrator.execute("do it", ctx, providers, tools, hooks)
    events = hooks.emitted_events
    event_names = [e[0] for e in events]
    assert "agent:tool_call_start" in event_names
    assert "agent:tool_call_end" in event_names

@pytest.mark.asyncio
async def test_tool_call_end_has_full_output():
    """TOOL_CALL_END event carries full untruncated output (EVENT-005)."""
    # ... verify event data includes "full_output" key
```

**Step 2: Run — expect FAIL**

**Step 3: Define event constants in `events.py`** mapping spec EventKinds to Amplifier hook event names. Use `hooks.emit()` at the correct points in the loop.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: event emission for session and tool lifecycle"
```

---

### Task 1.7: Error Handling and Graceful Shutdown

**Files:**
- Modify: `amplifier_module_loop_agent/__init__.py`
- Test: `tests/test_error_handling.py`

**Spec coverage:** ERR-001–013, SHUT-001–009, STOP-004–005

**Step 1: Write failing test**

```python
# tests/test_error_handling.py
import pytest
from amplifier_core.llm_errors import AuthenticationError, ContextLengthError, RateLimitError
from amplifier_module_loop_agent.state import SessionState

@pytest.mark.asyncio
async def test_auth_error_closes_session():
    """Authentication error → session CLOSED, no retry."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        provider_error=AuthenticationError("invalid key", provider="openai")
    )
    with pytest.raises(AuthenticationError):
        await orchestrator.execute("hello", ctx, providers, tools, hooks)
    assert orchestrator._state_machine.state == SessionState.CLOSED

@pytest.mark.asyncio
async def test_tool_error_returned_to_llm():
    """Tool execution error → error result sent to LLM, not exception."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        responses=[tool_response("bad_tool", {}), text_response("handled")],
        tool_error=RuntimeError("tool broke"),
    )
    result = await orchestrator.execute("use bad_tool", ctx, providers, tools, hooks)
    assert result == "handled"  # LLM recovered

@pytest.mark.asyncio
async def test_unknown_tool_returns_error():
    """Unknown tool name → error result, not exception."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        responses=[
            tool_response("nonexistent_tool", {}),
            text_response("ok, that tool doesn't exist"),
        ]
    )
    result = await orchestrator.execute("try it", ctx, providers, tools, hooks)
    assert "nonexistent_tool" not in [t for t in tools]  # confirm unknown
    assert result is not None  # LLM recovered
```

**Step 2: Run — expect FAIL**

**Step 3: Implement error handling**

In the main loop:
- Wrap `provider.complete()` with `try/except`:
  - `AuthenticationError`, `ContextLengthError` → `state_machine.fatal_error()`, re-raise
  - `RateLimitError`, `LLMTimeoutError`, `ProviderUnavailableError` → handled by provider retry (Phase 2 work)
  - `LLMError(retryable=True)` → emit error event, re-raise
- Tool execution: `try/except Exception` → return `ToolResult(success=False, error=str(e))`
- Unknown tool: return error result with `f"Unknown tool: {name}. Available tools: {list}"`
- Graceful shutdown: on abort signal or fatal error, cancel in-flight LLM, flush events, emit SESSION_END, transition to CLOSED

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: error handling with LLM recovery and graceful shutdown"
```

---

### Task 1.8: History-to-Messages Conversion

**Files:**
- Create: `amplifier_module_loop_agent/messages.py`
- Test: `tests/test_messages.py`

**Spec coverage:** LOOP-010, STEER-003, STEER-010

**Step 1: Write failing test**

```python
# tests/test_messages.py
from amplifier_module_loop_agent.turns import UserTurn, AssistantTurn, SteeringTurn, ToolResultsTurn
from amplifier_module_loop_agent.messages import convert_history_to_messages

def test_user_turn_becomes_user_message():
    turns = [UserTurn(content="hello")]
    msgs = convert_history_to_messages(turns)
    assert msgs[0]["role"] == "user"
    assert msgs[0]["content"] == "hello"

def test_steering_turn_becomes_user_message():
    """SteeringTurn is converted to user-role message (STEER-003, STEER-010)."""
    turns = [SteeringTurn(content="try differently")]
    msgs = convert_history_to_messages(turns)
    assert msgs[0]["role"] == "user"
    assert "try differently" in msgs[0]["content"]
```

**Step 2: Run — expect FAIL**

**Step 3: Implement `convert_history_to_messages()`** — maps each Turn type to the appropriate Amplifier message dict format.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: history-to-messages conversion for LLM requests"
```

---

## Phase 2: Tool Extensions

**Goal:** Extend existing Amplifier tools and create new ones needed by the coding agent loop.

**Parallelizable with Phases 1, 3, and 5.**

---

### Task 2.1: tool-apply-patch (New Module)

**Files:**
- Create: `amplifier-module-tool-apply-patch/amplifier_module_tool_apply_patch/__init__.py`
- Create: `amplifier-module-tool-apply-patch/amplifier_module_tool_apply_patch/parser.py`
- Create: `amplifier-module-tool-apply-patch/pyproject.toml`
- Test: `amplifier-module-tool-apply-patch/tests/test_parser.py`
- Test: `amplifier-module-tool-apply-patch/tests/test_apply.py`

**Spec coverage:** PATCH-001–011, OAI-002

**New repo:** `amplifier-module-tool-apply-patch/`

**Step 1: Write failing tests for the parser**

```python
# tests/test_parser.py
from amplifier_module_tool_apply_patch.parser import parse_v4a_patch

def test_add_file():
    patch = """*** Begin Patch
*** Add File: src/hello.py
+print("hello world")
*** End Patch
"""
    ops = parse_v4a_patch(patch)
    assert len(ops) == 1
    assert ops[0].operation == "add_file"
    assert ops[0].path == "src/hello.py"
    assert ops[0].content == 'print("hello world")\n'

def test_delete_file():
    patch = """*** Begin Patch
*** Delete File: old.py
*** End Patch
"""
    ops = parse_v4a_patch(patch)
    assert ops[0].operation == "delete_file"
    assert ops[0].path == "old.py"

def test_update_file_with_hunk():
    patch = """*** Begin Patch
*** Update File: src/main.py
@@ def main():
     print("old")
-    print("old")
+    print("new")
     return 0
*** End Patch
"""
    ops = parse_v4a_patch(patch)
    assert ops[0].operation == "update_file"
    assert len(ops[0].hunks) == 1

def test_multiple_hunks_in_update():
    # PATCH-011: single Update File block can contain multiple @@ hunks
    pass

def test_move_file():
    # PATCH-006: *** Move to: new_path
    pass
```

**Step 2: Run — expect FAIL**

**Step 3: Implement v4a patch parser**

Parse the grammar:
- `*** Begin Patch` / `*** End Patch` — envelope
- `*** Add File: path` + lines prefixed with `+`
- `*** Delete File: path`
- `*** Update File: path` + optional `*** Move to: new_path` + hunks
- `@@ context_hint` + context lines (space prefix) + deletions (`-`) + additions (`+`)

**Step 4: Run — expect PASS**

**Step 5: Write failing tests for the apply logic**

```python
# tests/test_apply.py
import tempfile, os
from amplifier_module_tool_apply_patch import ApplyPatchTool

@pytest.mark.asyncio
async def test_apply_add_file(tmp_path):
    tool = ApplyPatchTool(config={"working_dir": str(tmp_path)})
    result = await tool.execute({"patch": "*** Begin Patch\n*** Add File: hello.py\n+print('hi')\n*** End Patch\n"})
    assert result.success
    assert (tmp_path / "hello.py").read_text() == "print('hi')\n"

@pytest.mark.asyncio
async def test_apply_update_file(tmp_path):
    (tmp_path / "main.py").write_text("def main():\n    print('old')\n    return 0\n")
    tool = ApplyPatchTool(config={"working_dir": str(tmp_path)})
    patch = """*** Begin Patch
*** Update File: main.py
@@ def main():
     print('old')
-    print('old')
+    print('new')
     return 0
*** End Patch
"""
    result = await tool.execute({"patch": patch})
    assert result.success
    assert "print('new')" in (tmp_path / "main.py").read_text()
```

**Step 6: Implement the Tool class** following the standard Amplifier tool protocol (`name`, `description`, `input_schema`, `execute(input) -> ToolResult`).

**Step 7: Run all tests — expect PASS**

**Step 8: Commit**
```bash
git add -A && git commit -m "feat: tool-apply-patch module with v4a format parser"
```

---

### Task 2.2: tool-bash Extensions

**Files:**
- Modify: `amplifier-module-tool-bash/amplifier_module_tool_bash/__init__.py`
- Test: `amplifier-module-tool-bash/tests/test_extensions.py`

**Spec coverage:** TOOL-008–009, EXEC-008, EXEC-010–013, TIMEOUT-001–005

**Extensions needed:**
1. Add `timeout_ms` parameter to input schema (per-call override)
2. Add `description` parameter (optional, for observability)
3. Return `duration_ms` in output
4. Change SIGTERM grace period from 0.5s to 2s (spec says 2s)
5. Add env-var filtering (exclude `*_API_KEY`, `*_SECRET`, etc.)

**Step 1: Write failing tests**

```python
# tests/test_extensions.py
import pytest, time

@pytest.mark.asyncio
async def test_timeout_ms_parameter():
    """Per-call timeout_ms overrides default (TIMEOUT-001)."""
    tool = make_bash_tool(config={"timeout": 30})
    result = await tool.execute({"command": "sleep 2", "timeout_ms": 1000})
    assert not result.success
    assert "timed out" in str(result.output).lower()

@pytest.mark.asyncio
async def test_duration_ms_in_output():
    """Output includes duration_ms (EXEC-010)."""
    tool = make_bash_tool()
    result = await tool.execute({"command": "echo hello"})
    assert "duration_ms" in result.output

@pytest.mark.asyncio
async def test_env_var_filtering():
    """API keys filtered from child process environment (EXEC-011)."""
    import os
    os.environ["TEST_API_KEY"] = "secret"
    try:
        tool = make_bash_tool()
        result = await tool.execute({"command": "env | grep TEST_API_KEY || echo 'not found'"})
        assert "not found" in result.output.get("stdout", "")
    finally:
        del os.environ["TEST_API_KEY"]
```

**Step 2: Run — expect FAIL**

**Step 3: Implement extensions**

- Add `timeout_ms` and `description` to `input_schema` properties
- In `_run_command()`: if `timeout_ms` in input, use `timeout_ms / 1000` as timeout (capped at `max_command_timeout_ms / 1000`)
- Record `time.monotonic()` before/after, include `duration_ms` in result
- Change SIGTERM wait from `await asyncio.sleep(0.5)` to `await asyncio.sleep(2.0)`
- Add env filtering: filter `os.environ` before passing to subprocess, excluding patterns matching `*_API_KEY`, `*_SECRET`, `*_TOKEN`, `*_PASSWORD`, `*_CREDENTIAL` (case-insensitive), always keeping `PATH`, `HOME`, `USER`, `SHELL`, `LANG`, `TERM`, `TMPDIR`

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: bash tool extensions — timeout_ms, duration_ms, env filtering, 2s SIGTERM grace"
```

---

### Task 2.3: tool-report-outcome (New Module)

**Files:**
- Create: `amplifier-module-tool-report-outcome/amplifier_module_tool_report_outcome/__init__.py`
- Create: `amplifier-module-tool-report-outcome/pyproject.toml`
- Test: `amplifier-module-tool-report-outcome/tests/test_report_outcome.py`

**Purpose:** Allows the coding agent to report a structured outcome back to the pipeline. The pipeline orchestrator reads this to decide which edge to follow.

**Step 1: Write failing test**

```python
# tests/test_report_outcome.py
import pytest

@pytest.mark.asyncio
async def test_report_success():
    tool = make_report_tool()
    result = await tool.execute({
        "status": "success",
        "preferred_label": "tests_pass",
        "notes": "All 42 tests passing",
    })
    assert result.success
    assert result.output["status"] == "success"

@pytest.mark.asyncio
async def test_report_fail():
    tool = make_report_tool()
    result = await tool.execute({
        "status": "fail",
        "failure_reason": "3 tests still failing",
    })
    assert result.success  # Tool itself succeeds
    assert result.output["status"] == "fail"

@pytest.mark.asyncio
async def test_invalid_status_rejected():
    tool = make_report_tool()
    result = await tool.execute({"status": "invalid_value"})
    assert not result.success
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** — Simple tool that validates and stores structured outcome data. The pipeline orchestrator retrieves this from the session result.

Input schema:
```json
{
  "status": {"type": "string", "enum": ["success", "partial_success", "retry", "fail"]},
  "preferred_label": {"type": "string"},
  "suggested_next_ids": {"type": "array", "items": {"type": "string"}},
  "context_updates": {"type": "object"},
  "notes": {"type": "string"},
  "failure_reason": {"type": "string"}
}
```

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: tool-report-outcome for structured pipeline results"
```

---

### Task 2.4: hooks-tool-truncation (New Module)

**Files:**
- Create: `amplifier-module-hooks-tool-truncation/amplifier_module_hooks_tool_truncation/__init__.py`
- Create: `amplifier-module-hooks-tool-truncation/pyproject.toml`
- Test: `amplifier-module-hooks-tool-truncation/tests/test_truncation.py`

**Spec coverage:** TRUNC-001–015

**This is a hook module** that registers on `tool:post` and truncates tool output before it reaches the LLM, while preserving the full output in the event.

**Step 1: Write failing test**

```python
# tests/test_truncation.py
import pytest

def test_character_truncation_head_tail():
    from amplifier_module_hooks_tool_truncation import truncate_output
    output = "A" * 100_000  # 100k chars
    result = truncate_output(output, max_chars=30_000, mode="head_tail")
    assert len(result) <= 30_000
    assert "[WARNING: Tool output was truncated" in result
    assert result.startswith("A")  # head preserved
    assert result.endswith("A")  # tail preserved

def test_line_truncation_after_char():
    """Character truncation runs FIRST, then line truncation (TRUNC-008–009)."""
    lines = ["line " + str(i) + "x" * 100 for i in range(1000)]
    output = "\n".join(lines)
    result = truncate_output(output, max_chars=30_000, mode="head_tail",
                            max_lines=256)
    assert result.count("\n") <= 256

def test_default_limits_per_tool():
    from amplifier_module_hooks_tool_truncation import DEFAULT_CHAR_LIMITS
    assert DEFAULT_CHAR_LIMITS["read_file"] == 50_000
    assert DEFAULT_CHAR_LIMITS["shell"] == 30_000
    assert DEFAULT_CHAR_LIMITS["grep"] == 20_000
```

**Step 2: Run — expect FAIL**

**Step 3: Implement the hook**

The hook registers on `tool:post` at a high priority (runs early). It:
1. Checks if tool output exceeds the per-tool character limit
2. Applies character-based truncation first (head_tail or tail mode)
3. Applies line-based truncation second
4. Stores full output in event data (for TOOL_CALL_END event)
5. Returns `HookResult(action="modify", ...)` with truncated output

Uses `HookResult(action="modify")` to replace the tool output with the truncated version.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: hooks-tool-truncation with two-pass character+line truncation"
```

---

## Phase 3: Provider Profiles (Bundle Composition)

**Goal:** Create provider-specific tool + prompt bundles that the pipeline selects per-node.

**Parallelizable with Phases 1, 2, and 5.**

---

### Task 3.1: System Prompts

**Files:**
- Create: `attractor-profiles/prompts/system-openai.md`
- Create: `attractor-profiles/prompts/system-anthropic.md`
- Create: `attractor-profiles/prompts/system-gemini.md`
- Test: Manual review — these are markdown files

**Spec coverage:** PROMPT-001–011, OAI-004, ANT-004, GEM-003

**Step 1: Create system prompts** mirroring each provider's reference agent:

**system-openai.md** — mirrors codex-rs: identity, apply_patch format, shell @ 10s default, coding best practices
**system-anthropic.md** — mirrors Claude Code: identity, edit_file format (old_string must be unique), shell @ 120s, file operation preferences
**system-gemini.md** — mirrors gemini-cli: identity, GEMINI.md conventions, read_many_files capability

Each prompt includes the 5 layers (PROMPT-001): base instructions, environment context placeholder, tool descriptions, project docs placeholder, user instruction override.

**Step 2: Commit**
```bash
git add -A && git commit -m "feat: provider-specific system prompts mirroring reference agents"
```

---

### Task 3.2: Profile Bundle YAML Files

**Files:**
- Create: `attractor-profiles/profiles/attractor-openai.yaml`
- Create: `attractor-profiles/profiles/attractor-anthropic.yaml`
- Create: `attractor-profiles/profiles/attractor-gemini.yaml`

**Spec coverage:** PROF-001–008, OAI-001–006, ANT-001–006, GEM-001–005

**Reference:** Amplifier bundle YAML format from `amplifier-foundation`

**attractor-openai.yaml:**
```yaml
name: attractor-openai
description: OpenAI coding agent profile (codex-rs aligned)

providers:
  - module: provider-openai

orchestrator:
  module: loop-agent
  config:
    default_command_timeout_ms: 10000

tools:
  - module: tool-filesystem
    config:
      expose: [read_file, write_file]  # write_file for NEW files only
  - module: tool-apply-patch           # OpenAI-specific
  - module: tool-bash
    config:
      timeout: 10
      safety_profile: standard
  - module: tool-search
  - module: tool-delegate
  - module: tool-report-outcome

hooks:
  - module: hooks-tool-truncation

context:
  - path: prompts/system-openai.md
    role: system
```

**attractor-anthropic.yaml:**
```yaml
name: attractor-anthropic
description: Anthropic coding agent profile (Claude Code aligned)

providers:
  - module: provider-anthropic

orchestrator:
  module: loop-agent
  config:
    default_command_timeout_ms: 120000  # 120s for Anthropic

tools:
  - module: tool-filesystem            # Full: read, write, edit
  - module: tool-bash
    config:
      timeout: 120
      safety_profile: standard
  - module: tool-search
  - module: tool-delegate
  - module: tool-report-outcome

hooks:
  - module: hooks-tool-truncation

context:
  - path: prompts/system-anthropic.md
    role: system
```

**attractor-gemini.yaml:** Similar pattern with Gemini-specific tools and timeouts.

**Step 1: Create the YAML files**

**Step 2: Commit**
```bash
git add -A && git commit -m "feat: provider profile bundles for OpenAI, Anthropic, Gemini"
```

---

### Task 3.3: Environment Context Builder

**Files:**
- Create: `amplifier_module_loop_agent/environment.py`
- Test: `tests/test_environment.py`

**Spec coverage:** ENVCTX-001–002, GIT-001–002, PROJDOC-001–007

**Step 1: Write failing test**

```python
# tests/test_environment.py
from amplifier_module_loop_agent.environment import build_environment_context

def test_environment_context_has_required_fields():
    ctx = build_environment_context(working_dir="/tmp/test")
    assert "working_directory" in ctx or "<environment>" in ctx
    assert "platform" in ctx
    assert "date" in ctx

def test_project_doc_discovery(tmp_path):
    (tmp_path / "AGENTS.md").write_text("# Project rules")
    (tmp_path / "CLAUDE.md").write_text("# Claude specific")
    from amplifier_module_loop_agent.environment import discover_project_docs
    docs = discover_project_docs(str(tmp_path), provider_id="anthropic")
    assert "AGENTS.md" in [d["name"] for d in docs]
    assert "CLAUDE.md" in [d["name"] for d in docs]

def test_project_doc_filters_by_provider(tmp_path):
    (tmp_path / "AGENTS.md").write_text("universal")
    (tmp_path / "CLAUDE.md").write_text("anthropic only")
    (tmp_path / "GEMINI.md").write_text("gemini only")
    docs = discover_project_docs(str(tmp_path), provider_id="anthropic")
    names = [d["name"] for d in docs]
    assert "AGENTS.md" in names
    assert "CLAUDE.md" in names
    assert "GEMINI.md" not in names  # Filtered out
```

**Step 2: Run — expect FAIL**

**Step 3: Implement**

- `build_environment_context()` — creates the `<environment>` block with working dir, platform, OS, date, git branch/status
- `discover_project_docs()` — walks from git root to CWD, loads AGENTS.md (universal), CLAUDE.md (anthropic), GEMINI.md (gemini), .codex/instructions.md (openai). Applies 32KB budget with truncation.
- `build_git_context()` — snapshots branch, modified/untracked counts, last 5 commits

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: environment context builder with project doc discovery"
```

---

### Task 3.4: System Prompt Assembly

**Files:**
- Create: `amplifier_module_loop_agent/prompt.py`
- Test: `tests/test_prompt.py`

**Spec coverage:** PROMPT-001–007, LOOP-009

**Step 1: Write failing test**

```python
# tests/test_prompt.py
from amplifier_module_loop_agent.prompt import build_system_prompt

def test_system_prompt_has_five_layers():
    prompt = build_system_prompt(
        base_prompt="You are a coding agent.",
        environment_context="<environment>...</environment>",
        tool_descriptions="Available tools: read_file, write_file",
        project_docs="# AGENTS.md\nAlways write tests.",
        user_instructions="Focus on the auth module.",
    )
    assert "You are a coding agent" in prompt
    assert "<environment>" in prompt
    assert "read_file" in prompt
    assert "AGENTS.md" in prompt
    assert "Focus on the auth module" in prompt  # Last = highest priority
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** — Concatenates the 5 layers in order per PROMPT-001.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: 5-layer system prompt assembly"
```

---

## Phase 4: Steering, Loop Detection, and Subagents

**Goal:** Complete the advanced loop-agent features.

**Depends on:** Phases 1 + 2 + 3

---

### Task 4.1: Steering Queue

**Files:**
- Modify: `amplifier_module_loop_agent/__init__.py`
- Test: `tests/test_steering.py`

**Spec coverage:** STEER-001–010

**Step 1: Write failing test**

```python
# tests/test_steering.py
import asyncio, pytest

@pytest.mark.asyncio
async def test_steer_injects_message_between_rounds():
    """Steering message appears after current tool round (STEER-001)."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        responses=[
            tool_response("read_file", {"file_path": "test.py"}),
            # After tool execution, steering is drained
            text_response("I see the steering message, adjusting approach"),
        ]
    )
    # Queue a steering message before execution
    orchestrator.steer("Focus on the login module instead")
    result = await orchestrator.execute("analyze the code", ctx, providers, tools, hooks)
    # Verify SteeringTurn was added to history
    steering_turns = [t for t in orchestrator._history if hasattr(t, '__class__') and t.__class__.__name__ == 'SteeringTurn']
    assert len(steering_turns) >= 1

@pytest.mark.asyncio
async def test_followup_queue():
    """Follow-up messages processed after current input completes (STEER-005)."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        responses=[
            text_response("done with first"),
            text_response("done with followup"),
        ]
    )
    orchestrator.follow_up("Now also update the tests")
    result = await orchestrator.execute("update main.py", ctx, providers, tools, hooks)
    # The follow-up should have been processed
    assert providers["default"].complete.call_count >= 2
```

**Step 2: Run — expect FAIL**

**Step 3: Implement**

Add to `AgentOrchestrator`:
- `steer(message: str)` — appends to `steering_queue`
- `follow_up(message: str)` — appends to `followup_queue`
- `_drain_steering()` — dequeues all steering messages, creates SteeringTurn for each, emits STEERING_INJECTED event
- In the main loop: call `_drain_steering()` before first LLM call and after each tool round
- After loop exits: check followup_queue, dequeue and recursively process

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: steering and follow-up queues"
```

---

### Task 4.2: Loop Detection

**Files:**
- Create: `amplifier_module_loop_agent/loop_detection.py`
- Test: `tests/test_loop_detection.py`

**Spec coverage:** DETECT-001–009

**Step 1: Write failing test**

```python
# tests/test_loop_detection.py
from amplifier_module_loop_agent.loop_detection import detect_loop

def test_no_loop_with_varied_calls():
    signatures = [("read_file", "a"), ("write_file", "b"), ("grep", "c")]
    assert detect_loop(signatures, window_size=10) is False

def test_detects_pattern_length_1():
    """Same call repeated 10 times (DETECT-006)."""
    signatures = [("read_file", "abc123")] * 10
    assert detect_loop(signatures, window_size=10) is True

def test_detects_pattern_length_2():
    """Alternating pattern of 2 repeated 5 times."""
    signatures = [("read_file", "a"), ("write_file", "b")] * 5
    assert detect_loop(signatures, window_size=10) is True

def test_detects_pattern_length_3():
    """Pattern of 3 repeated."""
    sigs = [("read_file", "a"), ("grep", "b"), ("write_file", "c")] * 4
    # window_size must be divisible by pattern_len
    assert detect_loop(sigs[-12:], window_size=12) is True

def test_window_too_small_returns_false():
    """Fewer than window_size calls → no detection (DETECT-005)."""
    signatures = [("read_file", "a")] * 5
    assert detect_loop(signatures, window_size=10) is False
```

**Step 2: Run — expect FAIL**

**Step 3: Implement**

```python
def detect_loop(signatures: list[tuple[str, str]], window_size: int = 10) -> bool:
    if len(signatures) < window_size:
        return False
    recent = signatures[-window_size:]
    for pattern_len in [1, 2, 3]:
        if window_size % pattern_len != 0:
            continue
        chunks = [recent[i:i+pattern_len] for i in range(0, window_size, pattern_len)]
        if all(chunk == chunks[0] for chunk in chunks):
            return True
    return False
```

**Step 4: Run — expect PASS**

**Step 5: Wire into the main loop** — after each tool round, if `config.enable_loop_detection` is true, call `detect_loop()`. If detected, inject warning as SteeringTurn and emit LOOP_DETECTION event.

**Step 6: Commit**
```bash
git add -A && git commit -m "feat: loop detection with pattern matching on tool signatures"
```

---

### Task 4.3: Subagent Tools via Delegate

**Files:**
- Modify: `amplifier_module_loop_agent/__init__.py`
- Test: `tests/test_subagents.py`

**Spec coverage:** SUB-001–015

**The existing `tool-delegate` already handles spawn/resume.** The key adaptations for the spec's subagent model:

1. `spawn_agent` — maps to `delegate(agent="self", instruction=task)` with `working_dir` and `max_turns` support
2. `send_input` — maps to `delegate(session_id=agent_id, instruction=message)` (resume)
3. `wait` — already handled (delegate blocks until child completes)
4. `close_agent` — graceful close not directly supported; we track active sessions and cancel on shutdown

**Step 1: Write failing test**

```python
# tests/test_subagents.py
import pytest

@pytest.mark.asyncio
async def test_spawn_agent_uses_delegate():
    """spawn_agent maps to delegate tool with agent='self'."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        config={"max_subagent_depth": 1},
        responses=[
            tool_response("delegate", {"agent": "self", "instruction": "fix the bug"}),
            text_response("Bug fixed via subagent"),
        ]
    )
    # Verify delegate tool is available
    assert "delegate" in tools

@pytest.mark.asyncio
async def test_subagent_depth_limiting():
    """max_subagent_depth prevents infinite recursion (SUB-013)."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        config={"max_subagent_depth": 0},  # No subagents allowed
    )
    # delegate tool should be excluded or disabled
    assert "delegate" not in tools or tools["delegate"].config.get("features", {}).get("self_delegation", {}).get("max_depth", 1) == 0
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** — Configure the delegate tool's `features.self_delegation.max_depth` from `config.max_subagent_depth`. The existing delegate tool already handles depth tracking.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: subagent support via delegate tool with depth limiting"
```

---

### Task 4.4: Context Window Awareness

**Files:**
- Modify: `amplifier_module_loop_agent/__init__.py`
- Test: `tests/test_context_window.py`

**Spec coverage:** CTX-001–004

**Step 1: Write failing test**

```python
# tests/test_context_window.py
@pytest.mark.asyncio
async def test_context_window_warning_at_80_percent():
    """Emit warning when usage exceeds 80% of context window."""
    orchestrator, ctx, providers, tools, hooks = make_test_harness(
        config={"context_window_size": 1000},
    )
    # Simulate conversation that exceeds 80%
    # ... (mock message history that's ~800+ tokens via 4-char heuristic)
    await orchestrator.execute("hello", ctx, providers, tools, hooks)
    warnings = [e for e in hooks.emitted_events if "context_window" in e[0]]
    assert len(warnings) >= 1
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** — After converting history to messages, estimate tokens (1 token ≈ 4 chars) and emit warning if > 80% of context window. Informational only, no automatic compaction.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: context window awareness with 80% warning"
```

---

## Phase 5: Core loop-pipeline Orchestrator

**Goal:** A working pipeline engine that parses DOT graphs, walks the graph, and spawns coding agent sessions per node.

**New repo:** `amplifier-module-loop-pipeline/`

**Parallelizable with Phases 1–3.**

**Spec coverage:** All pipeline categories (DOT-*, GATTR-*, NATTR-*, EDGE-*, NTYPE-*, EXEC-*, ESEL-*, etc.)

---

### Task 5.1: Module Skeleton + Mount Function

**Files:**
- Create: `amplifier-module-loop-pipeline/amplifier_module_loop_pipeline/__init__.py`
- Create: `amplifier-module-loop-pipeline/pyproject.toml`
- Test: `tests/test_mount.py`

**Reference:** Same pattern as Task 1.1

**Step 1: Write failing test** — verify mount registers an orchestrator

**Step 2: Run — expect FAIL**

**Step 3: Implement mount** — reads DOT graph path/content from config

The orchestrator receives the DOT graph via config:
```python
config = {
    "dot_source": "digraph { ... }",    # Inline DOT source
    # OR
    "dot_file": "path/to/pipeline.dot",  # File path
}
```

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: loop-pipeline module skeleton"
```

---

### Task 5.2: DOT Parser

**Files:**
- Create: `amplifier_module_loop_pipeline/dot_parser.py`
- Test: `tests/test_dot_parser.py`

**Spec coverage:** DOT-001–017

**Step 1: Write failing tests**

```python
# tests/test_dot_parser.py
from amplifier_module_loop_pipeline.dot_parser import parse_dot

def test_simple_graph():
    graph = parse_dot("""
    digraph pipeline {
        start [shape=Mdiamond]
        plan [label="Plan the work"]
        implement [label="Implement"]
        done [shape=Msquare]
        start -> plan -> implement -> done
    }
    """)
    assert len(graph.nodes) == 4
    assert len(graph.edges) == 3
    assert graph.nodes["start"].shape == "Mdiamond"

def test_rejects_undirected_graph():
    with pytest.raises(ValueError, match="digraph"):
        parse_dot("graph { A -- B }")

def test_chained_edges_expanded():
    """A -> B -> C expands to A->B and B->C (DOT-009)."""
    graph = parse_dot("digraph { A -> B -> C }")
    assert len(graph.edges) == 2

def test_node_defaults():
    """node [...] sets baseline attributes (DOT-010)."""
    graph = parse_dot("""
    digraph {
        node [shape=box, max_retries=3]
        A
        B [shape=diamond]
    }
    """)
    assert graph.nodes["A"].shape == "box"
    assert graph.nodes["A"].attrs.get("max_retries") == 3
    assert graph.nodes["B"].shape == "diamond"  # Override

def test_subgraph_support():
    graph = parse_dot("""
    digraph {
        subgraph cluster_impl {
            label="Implementation"
            code
            test
        }
        code -> test
    }
    """)
    assert "code" in graph.nodes
    assert "test" in graph.nodes

def test_attribute_value_types():
    """String, Integer, Float, Boolean, Duration parsing (DOT-008)."""
    graph = parse_dot("""
    digraph {
        A [timeout=30s, max_retries=3, goal_gate=true, label="Step A"]
    }
    """)
    assert graph.nodes["A"].attrs["timeout"] == 30000  # 30s → ms
    assert graph.nodes["A"].attrs["max_retries"] == 3
    assert graph.nodes["A"].attrs["goal_gate"] is True

def test_comments_stripped():
    graph = parse_dot("""
    digraph {
        // This is a comment
        A -> B /* inline comment */
    }
    """)
    assert len(graph.nodes) == 2

def test_graph_level_attributes():
    graph = parse_dot("""
    digraph {
        goal="Build the feature"
        default_max_retry=5
        A [shape=Mdiamond]
        B [shape=Msquare]
        A -> B
    }
    """)
    assert graph.goal == "Build the feature"
    assert graph.default_max_retry == 5
```

**Step 2: Run — expect FAIL**

**Step 3: Implement DOT parser**

Build a parser that handles:
- `digraph` keyword (reject `graph`, `strict`)
- Node declarations with attributes
- Edge declarations with `->` (reject `--`)
- Chained edges: `A -> B -> C [attrs]`
- Attribute blocks: `[key=val, key=val]`
- Value types: string (quoted), integer, float, boolean, duration
- `node [...]` and `edge [...]` default blocks
- Subgraph blocks
- Comment stripping (`//` and `/* */`)
- Graph-level attributes

Output: `Graph` model with `nodes: dict[str, Node]`, `edges: list[Edge]`, graph-level attributes.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: DOT parser with full attribute and subgraph support"
```

---

### Task 5.3: Graph Model and Validation

**Files:**
- Create: `amplifier_module_loop_pipeline/graph.py`
- Create: `amplifier_module_loop_pipeline/validation.py`
- Test: `tests/test_validation.py`

**Spec coverage:** LINT-001–018, NTYPE-001–009, NATTR-001–017, EDGE-001–006

**Step 1: Write failing tests**

```python
# tests/test_validation.py
from amplifier_module_loop_pipeline.validation import validate, validate_or_raise

def test_missing_start_node():
    """ERROR: no start node (LINT-003)."""
    graph = make_graph(nodes={"a": box_node(), "exit": msquare_node()}, edges=[("a", "exit")])
    diags = validate(graph)
    assert any(d.severity == "ERROR" and d.rule == "start_node" for d in diags)

def test_missing_exit_node():
    graph = make_graph(nodes={"start": mdiamond_node(), "a": box_node()}, edges=[("start", "a")])
    diags = validate(graph)
    assert any(d.severity == "ERROR" and d.rule == "terminal_node" for d in diags)

def test_unreachable_node():
    graph = make_graph(
        nodes={"start": mdiamond_node(), "a": box_node(), "orphan": box_node(), "exit": msquare_node()},
        edges=[("start", "a"), ("a", "exit")]  # orphan not reachable
    )
    diags = validate(graph)
    assert any(d.rule == "reachability" for d in diags)

def test_edge_target_exists():
    graph = make_graph(
        nodes={"start": mdiamond_node()},
        edges=[("start", "nonexistent")]
    )
    diags = validate(graph)
    assert any(d.rule == "edge_target_exists" for d in diags)

def test_validate_or_raise():
    graph = make_graph(nodes={}, edges=[])
    with pytest.raises(Exception):  # ValidationError
        validate_or_raise(graph)

def test_valid_graph_passes():
    graph = make_graph(
        nodes={"start": mdiamond_node(), "work": box_node(), "exit": msquare_node()},
        edges=[("start", "work"), ("work", "exit")]
    )
    diags = validate(graph)
    errors = [d for d in diags if d.severity == "ERROR"]
    assert len(errors) == 0
```

**Step 2: Run — expect FAIL**

**Step 3: Implement**

`graph.py` — data models:
```python
@dataclass
class Node:
    id: str
    label: str
    shape: str = "box"
    type: str = ""
    prompt: str = ""
    attrs: dict[str, Any] = field(default_factory=dict)
    # Resolved properties
    handler_type: str = ""  # Resolved from type or shape

@dataclass
class Edge:
    from_node: str
    to_node: str
    label: str = ""
    condition: str = ""
    weight: int = 0
    attrs: dict[str, Any] = field(default_factory=dict)

@dataclass
class Graph:
    nodes: dict[str, Node]
    edges: list[Edge]
    goal: str = ""
    default_max_retry: int = 50
    model_stylesheet: str = ""
    # ... other graph-level attrs
```

`validation.py` — lint rules: start_node, terminal_node, reachability, edge_target_exists, start_no_incoming, exit_no_outgoing, condition_syntax, stylesheet_syntax, type_known, fidelity_valid, retry_target_exists, goal_gate_has_retry, prompt_on_llm_nodes.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: graph model and validation with lint rules"
```

---

### Task 5.4: Outcome Model and Context Store

**Files:**
- Create: `amplifier_module_loop_pipeline/outcome.py`
- Create: `amplifier_module_loop_pipeline/context.py`
- Test: `tests/test_outcome.py`
- Test: `tests/test_context.py`

**Spec coverage:** OUT-001–007, CTX-001–005

**Step 1: Write failing tests**

```python
# tests/test_outcome.py
from amplifier_module_loop_pipeline.outcome import Outcome, StageStatus

def test_success_outcome():
    o = Outcome(status=StageStatus.SUCCESS, preferred_label="tests_pass")
    assert o.status == StageStatus.SUCCESS

def test_fail_with_reason():
    o = Outcome(status=StageStatus.FAIL, failure_reason="3 tests failing")
    assert o.failure_reason == "3 tests failing"

# tests/test_context.py
from amplifier_module_loop_pipeline.context import PipelineContext

def test_context_set_get():
    ctx = PipelineContext()
    ctx.set("outcome", "success")
    assert ctx.get("outcome") == "success"

def test_context_snapshot():
    ctx = PipelineContext()
    ctx.set("key", "value")
    snap = ctx.snapshot()
    assert snap["key"] == "value"

def test_context_clone_is_isolated():
    ctx = PipelineContext()
    ctx.set("key", "original")
    clone = ctx.clone()
    clone.set("key", "modified")
    assert ctx.get("key") == "original"
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** — `Outcome` with StageStatus enum (SUCCESS, PARTIAL_SUCCESS, RETRY, FAIL, SKIPPED), `PipelineContext` as a thread-safe key-value store.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: outcome model and pipeline context store"
```

---

### Task 5.5: Edge Selection Algorithm

**Files:**
- Create: `amplifier_module_loop_pipeline/edge_selection.py`
- Test: `tests/test_edge_selection.py`

**Spec coverage:** ESEL-001–010

**Step 1: Write failing tests**

```python
# tests/test_edge_selection.py
from amplifier_module_loop_pipeline.edge_selection import select_edge
from amplifier_module_loop_pipeline.outcome import Outcome, StageStatus

def test_condition_matching_takes_priority():
    """Step 1: condition-matching edges first (ESEL-002)."""
    edges = [
        Edge("A", "B", condition="outcome=fail"),
        Edge("A", "C", label="success"),
    ]
    outcome = Outcome(status=StageStatus.FAIL)
    ctx = PipelineContext()
    selected = select_edge(edges, outcome, ctx)
    assert selected.to_node == "B"

def test_preferred_label_match():
    """Step 2: preferred_label match (ESEL-003)."""
    edges = [Edge("A", "B", label="tests_pass"), Edge("A", "C", label="tests_fail")]
    outcome = Outcome(status=StageStatus.SUCCESS, preferred_label="tests_pass")
    selected = select_edge(edges, outcome, PipelineContext())
    assert selected.to_node == "B"

def test_label_normalization():
    """Labels normalized: lowercase, strip accelerators (ESEL-004)."""
    edges = [Edge("A", "B", label="[Y] Tests Pass")]
    outcome = Outcome(status=StageStatus.SUCCESS, preferred_label="tests pass")
    selected = select_edge(edges, outcome, PipelineContext())
    assert selected.to_node == "B"

def test_weight_tiebreak():
    """Higher weight wins (ESEL-006)."""
    edges = [Edge("A", "B", weight=1), Edge("A", "C", weight=5)]
    selected = select_edge(edges, Outcome(status=StageStatus.SUCCESS), PipelineContext())
    assert selected.to_node == "C"

def test_lexical_tiebreak():
    """Equal weight → lexical order (ESEL-007)."""
    edges = [Edge("A", "zebra"), Edge("A", "alpha")]
    selected = select_edge(edges, Outcome(status=StageStatus.SUCCESS), PipelineContext())
    assert selected.to_node == "alpha"

def test_no_edges_returns_none():
    selected = select_edge([], Outcome(status=StageStatus.SUCCESS), PipelineContext())
    assert selected is None
```

**Step 2: Run — expect FAIL**

**Step 3: Implement the 5-step edge selection** per ESEL-001–010.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: 5-step edge selection algorithm"
```

---

### Task 5.6: Condition Expression Language

**Files:**
- Create: `amplifier_module_loop_pipeline/conditions.py`
- Test: `tests/test_conditions.py`

**Spec coverage:** CEXPR-001–011

**Step 1: Write failing tests**

```python
# tests/test_conditions.py
from amplifier_module_loop_pipeline.conditions import evaluate_condition

def test_outcome_equals():
    assert evaluate_condition("outcome=success", outcome_status="success", context={}) is True
    assert evaluate_condition("outcome=fail", outcome_status="success", context={}) is False

def test_not_equals():
    assert evaluate_condition("outcome!=fail", outcome_status="success", context={}) is True

def test_context_lookup():
    assert evaluate_condition("context.last_stage=plan", outcome_status="", context={"last_stage": "plan"}) is True

def test_and_clauses():
    assert evaluate_condition(
        "outcome=success && context.tests_pass=true",
        outcome_status="success",
        context={"tests_pass": "true"}
    ) is True

def test_empty_condition_is_true():
    assert evaluate_condition("", outcome_status="fail", context={}) is True

def test_missing_context_key_is_empty_string():
    assert evaluate_condition("context.missing=value", outcome_status="", context={}) is False
    assert evaluate_condition("context.missing=", outcome_status="", context={}) is True
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** — Parse `Key Operator Literal` clauses joined by `&&`.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: condition expression language for edge routing"
```

---

### Task 5.7: Handler Registry and Core Handlers

**Files:**
- Create: `amplifier_module_loop_pipeline/handlers/__init__.py`
- Create: `amplifier_module_loop_pipeline/handlers/start.py`
- Create: `amplifier_module_loop_pipeline/handlers/exit.py`
- Create: `amplifier_module_loop_pipeline/handlers/codergen.py`
- Create: `amplifier_module_loop_pipeline/handlers/conditional.py`
- Create: `amplifier_module_loop_pipeline/handlers/tool.py`
- Test: `tests/test_handlers.py`

**Spec coverage:** HAND-001–007, HSTART-001–002, HEXIT-001–003, CODER-001–011, COND-001, TOOL-001–004

**Step 1: Write failing tests**

```python
# tests/test_handlers.py
import pytest
from amplifier_module_loop_pipeline.outcome import StageStatus

@pytest.mark.asyncio
async def test_start_handler_returns_success():
    handler = StartHandler()
    outcome = await handler.execute(node, context, graph, logs_root)
    assert outcome.status == StageStatus.SUCCESS

@pytest.mark.asyncio
async def test_exit_handler_returns_success():
    handler = ExitHandler()
    outcome = await handler.execute(node, context, graph, logs_root)
    assert outcome.status == StageStatus.SUCCESS

@pytest.mark.asyncio
async def test_codergen_handler_calls_backend(tmp_path):
    """Codergen calls backend.run() and returns outcome."""
    backend = MockBackend(return_value="Implementation complete")
    handler = CodergenHandler(backend=backend)
    node = make_node(id="implement", prompt="Build the feature for $goal")
    graph = make_graph(goal="user auth")
    outcome = await handler.execute(node, context, graph, str(tmp_path))
    assert outcome.status == StageStatus.SUCCESS
    # Verify $goal was expanded in prompt
    assert "user auth" in backend.last_prompt

@pytest.mark.asyncio
async def test_codergen_writes_stage_files(tmp_path):
    """Codergen writes prompt.md, response.md, status.json (CODER-003–009)."""
    handler = CodergenHandler(backend=MockBackend("done"))
    await handler.execute(make_node(id="step1"), context, graph, str(tmp_path))
    assert (tmp_path / "step1" / "prompt.md").exists()
    assert (tmp_path / "step1" / "response.md").exists()
    assert (tmp_path / "step1" / "status.json").exists()

@pytest.mark.asyncio
async def test_conditional_handler_is_noop():
    handler = ConditionalHandler()
    outcome = await handler.execute(node, context, graph, logs_root)
    assert outcome.status == StageStatus.SUCCESS

@pytest.mark.asyncio
async def test_tool_handler_runs_command(tmp_path):
    node = make_node(id="lint", attrs={"tool_command": "echo 'hello'"})
    handler = ToolHandler()
    outcome = await handler.execute(node, context, graph, str(tmp_path))
    assert outcome.status == StageStatus.SUCCESS
    assert "hello" in context.get("tool.output", "")
```

**Step 2: Run — expect FAIL**

**Step 3: Implement handlers**

Each handler implements: `async execute(node, context, graph, logs_root) -> Outcome`

- **StartHandler:** Returns `Outcome(status=SUCCESS)` immediately
- **ExitHandler:** Returns `Outcome(status=SUCCESS)` immediately
- **CodergenHandler:** Expands `$goal`, writes prompt.md, calls `backend.run()`, writes response.md and status.json, synthesizes Outcome from response
- **ConditionalHandler:** Returns `Outcome(status=SUCCESS)` (routing handled by edge selection)
- **ToolHandler:** Reads `tool_command`, executes via subprocess, returns SUCCESS or FAIL

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: handler registry with start, exit, codergen, conditional, tool handlers"
```

---

### Task 5.8: Pipeline Execution Engine

**Files:**
- Modify: `amplifier_module_loop_pipeline/__init__.py`
- Test: `tests/test_execution.py`

**Spec coverage:** EXEC-001–018

This is the core of the pipeline orchestrator — the `execute()` method that walks the graph.

**Step 1: Write failing tests**

```python
# tests/test_execution.py
import pytest

@pytest.mark.asyncio
async def test_simple_linear_pipeline(tmp_path):
    """start → plan → implement → exit."""
    orchestrator = make_pipeline_orchestrator(
        dot_source="""
        digraph {
            start [shape=Mdiamond]
            plan [prompt="Plan the work"]
            implement [prompt="Build it"]
            exit [shape=Msquare]
            start -> plan -> implement -> exit
        }
        """,
        backend=MockBackend("done"),
        logs_root=str(tmp_path),
    )
    result = await orchestrator.execute("run", context, providers, tools, hooks)
    assert "done" in result.lower() or result  # Pipeline completed

@pytest.mark.asyncio
async def test_conditional_branching(tmp_path):
    """Condition-based routing."""
    orchestrator = make_pipeline_orchestrator(
        dot_source="""
        digraph {
            start [shape=Mdiamond]
            check [shape=diamond]
            pass_path [prompt="Tests pass"]
            fail_path [prompt="Tests fail"]
            exit [shape=Msquare]
            start -> check
            check -> pass_path [condition="outcome=success"]
            check -> fail_path [condition="outcome=fail"]
            pass_path -> exit
            fail_path -> exit
        }
        """,
        backend=MockBackend("done"),
        logs_root=str(tmp_path),
    )
    result = await orchestrator.execute("run", context, providers, tools, hooks)
    assert result is not None
```

**Step 2: Run — expect FAIL**

**Step 3: Implement the execution engine**

Core loop (from spec EXEC-005):
```
parse DOT → validate → initialize context → resolve start node

current_node = start_node
while True:
    if current_node is terminal:
        check_goal_gates()  # May redirect
        break

    handler = resolve_handler(current_node)
    outcome = retry_policy.execute(handler, current_node, context)

    completed_nodes.append(current_node)
    node_outcomes[current_node.id] = outcome
    apply_context_updates(outcome)
    save_checkpoint()

    next_edge = select_edge(outgoing_edges, outcome, context)
    if next_edge is None:
        if outcome.status == FAIL:
            raise PipelineError("failed with no outgoing fail edge")
        break

    if next_edge.loop_restart:
        restart_run(next_edge.to_node)

    current_node = graph.nodes[next_edge.to_node]

return last_outcome
```

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: pipeline execution engine with graph walking"
```

---

### Task 5.9: CodergenBackend Adapter (Sessions All The Way Down)

**Files:**
- Create: `amplifier_module_loop_pipeline/backend.py`
- Test: `tests/test_backend.py`

**This is the critical integration point** — where the pipeline spawns coding agent sessions.

**Step 1: Write failing test**

```python
# tests/test_backend.py
import pytest
from amplifier_module_loop_pipeline.backend import AmplifierBackend

@pytest.mark.asyncio
async def test_backend_spawns_session():
    """Backend uses coordinator.get_capability('session.spawn') to create child session."""
    coordinator = make_mock_coordinator(spawn_returns={"output": "done", "session_id": "child-1"})
    backend = AmplifierBackend(coordinator, profiles={"anthropic": "attractor-anthropic"})
    node = make_node(id="implement", attrs={"llm_provider": "anthropic", "llm_model": "claude-sonnet-4-6"})
    result = await backend.run(node, "Build the feature", PipelineContext())
    assert coordinator.spawn_called
    assert "done" in str(result)

@pytest.mark.asyncio
async def test_backend_selects_profile_by_provider():
    """Different providers → different profile bundles."""
    coordinator = make_mock_coordinator(spawn_returns={"output": "ok", "session_id": "c-1"})
    backend = AmplifierBackend(coordinator, profiles={
        "anthropic": "attractor-anthropic",
        "openai": "attractor-openai",
    })
    node_anthropic = make_node(attrs={"llm_provider": "anthropic"})
    node_openai = make_node(attrs={"llm_provider": "openai"})

    await backend.run(node_anthropic, "task", PipelineContext())
    first_profile = coordinator.last_spawn_kwargs["child_bundle"]

    await backend.run(node_openai, "task", PipelineContext())
    second_profile = coordinator.last_spawn_kwargs["child_bundle"]

    assert first_profile != second_profile

@pytest.mark.asyncio
async def test_backend_extracts_outcome_from_response():
    """If child returns JSON outcome, parse it. Otherwise wrap in SUCCESS."""
    coordinator = make_mock_coordinator(spawn_returns={
        "output": '{"status": "fail", "failure_reason": "3 tests failing"}',
        "session_id": "c-1"
    })
    backend = AmplifierBackend(coordinator, profiles={"anthropic": "attractor-anthropic"})
    result = await backend.run(make_node(attrs={"llm_provider": "anthropic"}), "task", PipelineContext())
    # Should be an Outcome, not a raw string
    assert hasattr(result, 'status')
    assert result.status == StageStatus.FAIL
```

**Step 2: Run — expect FAIL**

**Step 3: Implement the backend adapter**

```python
class AmplifierBackend:
    """CodergenBackend implementation using Amplifier session spawning."""

    def __init__(self, coordinator, profiles: dict[str, str]):
        self._coordinator = coordinator
        self._profiles = profiles  # {"anthropic": "attractor-anthropic", ...}
        self._spawn_fn = None

    async def run(self, node: Node, prompt: str, context: PipelineContext) -> str | Outcome:
        if self._spawn_fn is None:
            self._spawn_fn = self._coordinator.get_capability("session.spawn")

        provider = node.attrs.get("llm_provider", "anthropic")
        model = node.attrs.get("llm_model")
        profile_name = self._profiles.get(provider)

        result = await self._spawn_fn(
            agent_name=profile_name,
            instruction=prompt,
            orchestrator_config={
                "reasoning_effort": node.attrs.get("reasoning_effort", "high"),
            },
            provider_preferences=[{"provider": provider, "model": model}] if model else None,
        )

        # Try to parse structured outcome from response
        response_text = result.get("output", "")
        outcome = self._try_parse_outcome(response_text)
        if outcome:
            return outcome
        return response_text  # Codergen handler wraps this in SUCCESS
```

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: CodergenBackend adapter using Amplifier session spawning"
```

---

### Task 5.10: Retry Logic

**Files:**
- Create: `amplifier_module_loop_pipeline/retry.py`
- Test: `tests/test_retry.py`

**Spec coverage:** RETRY-001–011, FAIL-001

**Step 1: Write failing tests**

```python
# tests/test_retry.py
import pytest
from amplifier_module_loop_pipeline.retry import RetryPolicy, execute_with_retry
from amplifier_module_loop_pipeline.outcome import Outcome, StageStatus

@pytest.mark.asyncio
async def test_success_on_first_try():
    handler = MockHandler(outcomes=[Outcome(status=StageStatus.SUCCESS)])
    policy = RetryPolicy(max_attempts=3)
    result = await execute_with_retry(handler, node, context, graph, logs, policy)
    assert result.status == StageStatus.SUCCESS
    assert handler.call_count == 1

@pytest.mark.asyncio
async def test_retry_on_retry_outcome():
    handler = MockHandler(outcomes=[
        Outcome(status=StageStatus.RETRY),
        Outcome(status=StageStatus.RETRY),
        Outcome(status=StageStatus.SUCCESS),
    ])
    policy = RetryPolicy(max_attempts=3)
    result = await execute_with_retry(handler, node, context, graph, logs, policy)
    assert result.status == StageStatus.SUCCESS
    assert handler.call_count == 3

@pytest.mark.asyncio
async def test_fail_not_retried():
    """FAIL outcome returns immediately (RETRY-006)."""
    handler = MockHandler(outcomes=[Outcome(status=StageStatus.FAIL, failure_reason="bad")])
    policy = RetryPolicy(max_attempts=3)
    result = await execute_with_retry(handler, node, context, graph, logs, policy)
    assert result.status == StageStatus.FAIL
    assert handler.call_count == 1

@pytest.mark.asyncio
async def test_allow_partial_on_exhaustion():
    """allow_partial=true → PARTIAL_SUCCESS after retries exhausted (RETRY-005)."""
    handler = MockHandler(outcomes=[Outcome(status=StageStatus.RETRY)] * 3)
    policy = RetryPolicy(max_attempts=3)
    node = make_node(attrs={"allow_partial": True})
    result = await execute_with_retry(handler, node, context, graph, logs, policy)
    assert result.status == StageStatus.PARTIAL_SUCCESS
```

**Step 2: Run — expect FAIL**

**Step 3: Implement** per RETRY-001–011 with exponential backoff.

**Step 4: Run — expect PASS**

**Step 5: Commit**
```bash
git add -A && git commit -m "feat: retry logic with exponential backoff and allow_partial"
```

---

## Phase 6: Full Pipeline Features

**Goal:** Complete the remaining pipeline capabilities.

**Depends on:** Phases 4 + 5

---

### Task 6.1: Model Stylesheet

**Files:**
- Create: `amplifier_module_loop_pipeline/stylesheet.py`
- Test: `tests/test_stylesheet.py`

**Spec coverage:** STYLE-001–007

Parse CSS-like stylesheet and apply to nodes:
```
* { llm_model: claude-sonnet-4-6; llm_provider: anthropic; }
.code { llm_model: claude-opus-4-6; }
#critical_review { llm_provider: openai; llm_model: gpt-5.2; reasoning_effort: high; }
```

Implement selector matching (universal, class, ID), specificity ordering, and the resolution chain (node attribute > stylesheet > graph default > system default).

---

### Task 6.2: Context Fidelity Modes

**Files:**
- Create: `amplifier_module_loop_pipeline/fidelity.py`
- Test: `tests/test_fidelity.py`

**Spec coverage:** FID-001–010

Implement the 6 fidelity modes (full, truncate, compact, summary:low/medium/high) and the resolution precedence chain (edge > node > graph > default:compact). The `full` mode reuses sessions via thread_id mapping.

---

### Task 6.3: Goal Gate Enforcement

**Files:**
- Modify: `amplifier_module_loop_pipeline/__init__.py`
- Test: `tests/test_goal_gates.py`

**Spec coverage:** GOAL-001–006

When the pipeline reaches the exit node, check all visited nodes with `goal_gate=true`. If any failed, jump to their retry_target instead of exiting.

---

### Task 6.4: Parallel and Fan-In Handlers

**Files:**
- Create: `amplifier_module_loop_pipeline/handlers/parallel.py`
- Create: `amplifier_module_loop_pipeline/handlers/fan_in.py`
- Test: `tests/test_parallel.py`

**Spec coverage:** PAR-001–013, FANIN-001–005, CONC-001–004

Parallel handler: spawns branches with cloned contexts, respects join_policy (wait_all, first_success, k_of_n, quorum) and error_policy (fail_fast, continue, ignore), bounded by max_parallel.

Fan-in handler: reads `parallel.results`, ranks candidates, selects winner.

---

### Task 6.5: Wait For Human Handler

**Files:**
- Create: `amplifier_module_loop_pipeline/handlers/human.py`
- Create: `amplifier_module_loop_pipeline/interviewer.py`
- Test: `tests/test_human.py`

**Spec coverage:** HUMAN-001–008, INTV-001–010

Maps to Amplifier's existing approval system. Derives choices from outgoing edges, presents via the interviewer interface (which maps to Amplifier's ApprovalProvider).

---

### Task 6.6: Manager Loop Handler

**Files:**
- Create: `amplifier_module_loop_pipeline/handlers/manager_loop.py`
- Test: `tests/test_manager_loop.py`

**Spec coverage:** MGR-001–010, COMP-001–002

Supervisor/child pipeline pattern. The manager spawns a child pipeline and observes/steers it.

---

### Task 6.7: Checkpointing and Resume

**Files:**
- Create: `amplifier_module_loop_pipeline/checkpoint.py`
- Test: `tests/test_checkpoint.py`

**Spec coverage:** CHKP-001–006

Save checkpoint.json after each node completes. Support resume from checkpoint with context restoration and retry counter restoration.

---

### Task 6.8: Transforms (Variable Expansion, Stylesheet Application)

**Files:**
- Create: `amplifier_module_loop_pipeline/transforms.py`
- Test: `tests/test_transforms.py`

**Spec coverage:** XFORM-001–006

Built-in transforms: variable expansion (`$goal`), stylesheet application, preamble synthesis for non-full fidelity.

---

### Task 6.9: Artifact Store

**Files:**
- Create: `amplifier_module_loop_pipeline/artifacts.py`
- Test: `tests/test_artifacts.py`

**Spec coverage:** ART-001–004

Named, typed storage with file-backing threshold (100KB).

---

### Task 6.10: Pipeline Events

**Files:**
- Modify: `amplifier_module_loop_pipeline/__init__.py`
- Test: `tests/test_pipeline_events.py`

**Spec coverage:** EVT-001–008

Emit PipelineStarted, PipelineCompleted, PipelineFailed, StageStarted, StageCompleted, StageFailed, StageRetrying, CheckpointSaved, etc.

---

### Task 6.11: Run Directory Structure

**Files:**
- Modify: `amplifier_module_loop_pipeline/handlers/codergen.py`
- Test: `tests/test_run_directory.py`

**Spec coverage:** DIR-001, STAT-001–004

Each execution produces: `checkpoint.json`, `manifest.json`, per-node subdirectories with `status.json`/`prompt.md`/`response.md`, `artifacts/` directory.

---

## Phase 7: Integration, Parity, and Interop

**Goal:** End-to-end validation across all providers and interop with the spec's DOT file format.

**Depends on:** Phase 6

---

### Task 7.1: Cross-Provider Parity Matrix

**Files:**
- Create: `tests/integration/test_parity_matrix.py`

**Spec coverage:** TEST-001–016

Run each test scenario across OpenAI, Anthropic, and Gemini profiles:

| Test | OpenAI | Anthropic | Gemini |
|------|--------|-----------|--------|
| Simple file creation | | | |
| Read + edit file | | | |
| Multi-file edit | | | |
| Shell command | | | |
| Shell timeout | | | |
| Grep + glob | | | |
| Multi-step task | | | |
| Truncation | | | |
| Parallel tool calls | | | |
| Steering | | | |
| Reasoning effort | | | |
| Subagent spawn + wait | | | |
| Loop detection | | | |
| Error recovery | | | |
| Provider-specific edit format | | | |

---

### Task 7.2: Integration Smoke Tests

**Files:**
- Create: `tests/integration/test_smoke.py`

**Spec coverage:** TEST-017–024

End-to-end tests with real API keys:
1. Simple file creation → assert file exists
2. Read and edit → assert file modified
3. Shell execution → verify tool call in events
4. Truncation → verify TOOL_CALL_END has full output, ToolResult has marker
5. Steering → verify agent adjusts
6. Subagent → verify subagent calls in events
7. Timeout → verify graceful handling

---

### Task 7.3: Pipeline End-to-End Test

**Files:**
- Create: `tests/integration/test_pipeline_e2e.py`

Run a complete pipeline from DOT file through to completion:

```python
@pytest.mark.asyncio
async def test_full_pipeline():
    """A 3-stage pipeline: plan → implement → test."""
    dot = """
    digraph {
        goal="Add a greeting function"
        start [shape=Mdiamond]
        plan [prompt="Plan how to add a greeting function to $goal"]
        implement [prompt="Implement the plan"]
        test [prompt="Write and run tests"]
        exit [shape=Msquare]
        start -> plan -> implement -> test -> exit
        test -> implement [condition="outcome=fail", label="tests_fail"]
    }
    """
    # Run with mock backend or real providers
    result = await run_pipeline(dot, providers=["anthropic"])
    assert result.status in (StageStatus.SUCCESS, StageStatus.PARTIAL_SUCCESS)
```

---

### Task 7.4: DOT File Interop Validation

**Files:**
- Create: `tests/integration/test_interop.py`

Validate that DOT files from the Attractor spec parse and execute correctly. Test with example graphs from the spec. Ensure our pipeline can consume DOT files produced by other Attractor implementations and vice versa.

---

### Task 7.5: Shadow Environment Validation

Run the full test suite in a shadow environment with all local sources (loop-agent, loop-pipeline, tool-apply-patch, hooks-tool-truncation, tool-report-outcome + all existing modules) to verify integration.

---

## Risks and Open Questions

### Must Resolve During Implementation

| # | Question | Phase | Impact |
|---|----------|-------|--------|
| R1 | **Orchestrator protocol returns `str`** — the pipeline needs structured `Outcome` from coding agents, but `execute()` returns `str`. How does the backend extract structured outcomes? | 5.9 | The coding agent uses tool-report-outcome to write a JSON blob. The backend parses this from the response text or reads it from a well-known event. |
| R2 | **Provider-mock on pipeline session** — the pipeline orchestrator doesn't call LLMs directly for most operations, but the kernel requires a mounted provider. Use provider-mock, or optionally a real provider for edge classification. | 5.1 | Use provider-mock by default. Add optional real provider for LLM-based edge classification in fan-in. |
| R3 | **spawn API bundle resolution** — how does the pipeline resolve profile bundle names to actual bundle objects? | 5.9 | The spawn capability accepts agent names that resolve via the coordinator's agent registry. Profile bundles are registered as agents. |
| R4 | **Session reuse for `full` fidelity** — the spec says nodes sharing a thread_id reuse the same LLM session. Amplifier sessions are created per-spawn. | 6.2 | Maintain a session pool in the backend. Reuse session_id for same thread_id. Use delegate's `session_id` parameter for resume. |
| R5 | **DOT parser complexity** — the spec's DOT grammar is substantial. Consider using an existing Python DOT parser (pydot, graphviz) and extending it. | 5.2 | Evaluate pydot for basic parsing, extend with custom attribute handling. If too limited, write custom parser. |

### Monitor During Implementation

| # | Risk | Mitigation |
|---|------|------------|
| M1 | Spawn overhead — spawning a new session per pipeline node may be slow | Profile. Consider session pooling for `full` fidelity. batch-spawn for parallel nodes. |
| M2 | Context loss between nodes — each spawn starts fresh | Fidelity modes (compact, summary) synthesize context. Full fidelity reuses sessions. |
| M3 | Error propagation through spawn → pipeline → parent | Each layer must correctly surface errors. The Phase 2 error-propagation fix helps. |
| M4 | DOT file portability — other implementations may produce slightly different DOT | Test with spec examples. Document supported subset. |

---

## Summary: Module Inventory

| Module | Type | New/Extend | Phase |
|--------|------|------------|-------|
| `amplifier-module-loop-agent` | Orchestrator | **New** | 1, 4 |
| `amplifier-module-loop-pipeline` | Orchestrator | **New** | 5, 6 |
| `amplifier-module-tool-apply-patch` | Tool | **New** | 2 |
| `amplifier-module-tool-report-outcome` | Tool | **New** | 2 |
| `amplifier-module-hooks-tool-truncation` | Hook | **New** | 2 |
| `amplifier-module-tool-bash` | Tool | **Extend** | 2 |
| `amplifier-module-tool-filesystem` | Tool | Unchanged | — |
| `amplifier-module-tool-search` | Tool | Unchanged | — |
| `amplifier-foundation` (tool-delegate) | Tool | Unchanged | — |
| `attractor-profiles/` | Bundles + Prompts | **New** | 3 |

**Total new modules: 5** | **Extended: 1** | **Unchanged: 4**

---

## Spec Requirement Coverage Map

### coding-agent-loop-spec.md (155 requirements)

| Category | Requirements | Phase |
|----------|-------------|-------|
| Architecture (ARCH) | 10 | 1 |
| Session State (SESS) | 17 | 1 |
| Configuration (CFG) | 9 | 1 |
| Turn Types (TURN) | 5 | 1 |
| Core Loop (LOOP) | 23 | 1 |
| Stop Conditions (STOP) | 5 | 1 |
| Steering (STEER) | 10 | 4 |
| Reasoning Effort (REASON) | 7 | 1 (already done in Phase 2 providers) |
| Events (EVENT) | 9 | 1 |
| Loop Detection (DETECT) | 9 | 4 |
| Provider Profiles (PROF) | 8 | 3 |
| OpenAI Profile (OAI) | 6 | 2 + 3 |
| Anthropic Profile (ANT) | 6 | 3 |
| Gemini Profile (GEM) | 5 | 3 |
| Shared Core Tools (TOOL) | 11 | 2 (existing tools) |
| Apply Patch (PATCH) | 11 | 2 |
| Tool Registry (REG) | 14 | 1 (uses Amplifier's tool system) |
| Execution Environment (EXEC) | 17 | 2 (tool-bash extensions) |
| Truncation (TRUNC) | 15 | 2 |
| Command Timeouts (TIMEOUT) | 5 | 2 |
| Context Window (CTX) | 4 | 4 |
| System Prompts (PROMPT) | 11 | 3 |
| Environment Context (ENVCTX) | 2 | 3 |
| Git Context (GIT) | 2 | 3 |
| Project Docs (PROJDOC) | 7 | 3 |
| Subagents (SUB) | 15 | 4 |
| Error Handling (ERR) | 13 | 1 |
| Graceful Shutdown (SHUT) | 9 | 1 |
| Cross-Provider Tests (TEST) | 24 | 7 |

### attractor-spec.md (281 requirements)

| Category | Requirements | Phase |
|----------|-------------|-------|
| DOT Parsing (DOT) | 17 | 5 |
| Graph Attributes (GATTR) | 7 | 5 |
| Node Attributes (NATTR) | 17 | 5 |
| Edge Attributes (EDGE) | 6 | 5 |
| Node Types (NTYPE) | 9 | 5 |
| Execution Engine (EXEC) | 18 | 5 |
| Edge Selection (ESEL) | 10 | 5 |
| Goal Gates (GOAL) | 6 | 6 |
| Retry Logic (RETRY) | 11 | 5 |
| Failure Routing (FAIL) | 1 | 5 |
| Concurrency (CONC) | 4 | 6 |
| Handler Interface (HAND) | 7 | 5 |
| Start Handler (HSTART) | 2 | 5 |
| Exit Handler (HEXIT) | 3 | 5 |
| Codergen Handler (CODER) | 11 | 5 |
| Backend Interface (BACK) | 2 | 5 |
| Human Handler (HUMAN) | 8 | 6 |
| Conditional Handler (COND) | 1 | 5 |
| Parallel Handler (PAR) | 13 | 6 |
| Fan-In Handler (FANIN) | 5 | 6 |
| Tool Handler (TOOL) | 4 | 5 |
| Manager Loop (MGR) | 10 | 6 |
| Context Store (CTX) | 5 | 5 |
| Outcome Model (OUT) | 7 | 5 |
| Checkpointing (CHKP) | 6 | 6 |
| Context Fidelity (FID) | 10 | 6 |
| Artifact Store (ART) | 4 | 6 |
| Run Directory (DIR) | 1 | 6 |
| Status Files (STAT) | 4 | 5 |
| Interviewer (INTV) | 10 | 6 |
| Validation/Linting (LINT) | 18 | 5 |
| Model Stylesheet (STYLE) | 7 | 6 |
| Condition Language (CEXPR) | 12 | 5 |
| Transforms (XFORM) | 6 | 6 |
| Observability (EVT) | 8 | 6 |
| Tool Hooks (HOOK) | 4 | 6 |
| HTTP Server (HTTP) | 3 | Future (SHOULD, not MUST) |
| Composition (COMP) | 2 | 6 |
| Error Categories (ERR) | 3 | 5 |

---

Plan complete and saved to `docs/plans/attractor-implementation-plan.md`.

**Execution options:**

1. **Subagent-Driven (this session)**
   - Fresh agent per task
   - Two-stage review (spec then quality)
   - Fast iteration

2. **Parallel Session**
   - Open new session for execution
   - Batch execution with human checkpoints

Which approach?
