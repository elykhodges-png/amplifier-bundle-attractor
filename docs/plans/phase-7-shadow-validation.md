# Phase 7: Shadow Environment Validation Plan

This document describes how to validate the full Attractor stack (loop-agent + loop-pipeline) in a shadow environment where both modules are installed from local source alongside amplifier-core and amplifier-foundation.

## Prerequisites

Before running shadow validation, all Phase 1-7 work must be committed and pushed:

- `amplifier-core` — kernel vocabulary (Phase 1 prereq) + provider improvements (Phase 2 prereq)
- `amplifier-module-loop-agent` — Phases 1-4 + Phase 7 (Tasks 7.1, 7.2)
- `amplifier-module-loop-pipeline` — Phases 5-6 + Phase 7 (Tasks 7.3, 7.4)
- `amplifier-foundation` — for bundle/agent infrastructure (used by pipeline to spawn coding agents)

## Repos Required in Shadow

| Repo | Role | Local Source Path |
|------|------|-------------------|
| `amplifier-core` | Kernel: message models, LLM errors, provider protocol | `../amplifier-core` |
| `amplifier-module-loop-agent` | Coding agent orchestrator | `../amplifier-module-loop-agent` |
| `amplifier-module-loop-pipeline` | Pipeline orchestrator (DOT graph engine) | `../amplifier-module-loop-pipeline` |
| `amplifier-foundation` | Bundle/agent infrastructure | `../amplifier-foundation` |
| `amplifier-module-provider-anthropic` | Anthropic provider (for live tests) | `../amplifier-module-provider-anthropic` |
| `amplifier-module-provider-openai` | OpenAI provider (for live tests) | `../amplifier-module-provider-openai` |
| `amplifier-module-provider-gemini` | Gemini provider (for live tests) | `../amplifier-module-provider-gemini` |

## Validation Steps

### Step 1: Create Shadow Environment

```bash
# From the attractor-next workspace root
shadow-operator create \
  --local-source amplifier-core=../amplifier-core \
  --local-source amplifier-module-loop-agent=../amplifier-module-loop-agent \
  --local-source amplifier-module-loop-pipeline=../amplifier-module-loop-pipeline
```

### Step 2: Run Unit Tests in Shadow

Verify all unit tests pass with the local sources installed together:

```bash
# loop-agent tests (250+ tests)
cd amplifier-module-loop-agent
uv run pytest tests/ -v
# Expected: all tests pass, including:
#   - test_parity_matrix.py (54 tests, 15 scenarios × 3 providers)
#   - test_integration_smoke.py (8 tests, 7-step smoke + full sequence)

# loop-pipeline tests (453+ tests)
cd amplifier-module-loop-pipeline
uv run pytest tests/ -v
# Expected: all tests pass, including:
#   - test_pipeline_e2e.py (27 tests, 3 fixtures + spec smoke test)
#   - test_dot_interop.py (65 tests, 5 spec DOT examples)
```

### Step 3: Verify Module Mounting

Verify both modules mount correctly in an Amplifier session:

```python
# Test script: verify_mount.py
import asyncio
from unittest.mock import AsyncMock, MagicMock

async def verify_loop_agent():
    """Verify loop-agent mounts as an orchestrator."""
    from amplifier_module_loop_agent import mount
    coordinator = MagicMock()
    coordinator.mount = AsyncMock()
    await mount(coordinator, config={})
    coordinator.mount.assert_called_once()
    args = coordinator.mount.call_args
    assert args[0][0] == "orchestrator"
    print("loop-agent: mount OK")

async def verify_loop_pipeline():
    """Verify loop-pipeline mounts as an orchestrator."""
    from amplifier_module_loop_pipeline import mount
    coordinator = MagicMock()
    coordinator.mount = AsyncMock()
    await mount(coordinator, config={})
    coordinator.mount.assert_called_once()
    args = coordinator.mount.call_args
    assert args[0][0] == "orchestrator"
    print("loop-pipeline: mount OK")

async def main():
    await verify_loop_agent()
    await verify_loop_pipeline()
    print("All mount checks passed.")

asyncio.run(main())
```

```bash
uv run python verify_mount.py
# Expected: "All mount checks passed."
```

### Step 4: Verify Cross-Module Import Compatibility

Verify that amplifier-core types used by both modules are compatible:

```python
# Test script: verify_imports.py
from amplifier_core.message_models import ChatRequest, ChatResponse, ToolCall, Usage, TextBlock, ThinkingBlock, ToolSpec, Message
from amplifier_core.llm_errors import LLMError
from amplifier_core.models import ToolResult

from amplifier_module_loop_agent import AgentOrchestrator
from amplifier_module_loop_agent.state import SessionState, SessionStateMachine
from amplifier_module_loop_agent.config import SessionConfig
from amplifier_module_loop_agent.events import AGENT_SESSION_START, AGENT_SESSION_END

from amplifier_module_loop_pipeline import PipelineOrchestrator
from amplifier_module_loop_pipeline.dot_parser import parse_dot
from amplifier_module_loop_pipeline.engine import PipelineEngine
from amplifier_module_loop_pipeline.graph import Graph, Node, Edge
from amplifier_module_loop_pipeline.outcome import Outcome, StageStatus
from amplifier_module_loop_pipeline.validation import validate, validate_or_raise
from amplifier_module_loop_pipeline.pipeline_events import PIPELINE_START, PIPELINE_COMPLETE

print("All imports successful — no version conflicts.")
```

```bash
uv run python verify_imports.py
# Expected: "All imports successful — no version conflicts."
```

### Step 5: Pipeline-to-Agent Interop (Mock)

Verify that the pipeline can conceptually spawn a coding agent session. In the full system, the pipeline's CodergenHandler calls a backend that spawns an Amplifier session running the loop-agent orchestrator. This test verifies the interface compatibility:

```python
# Test script: verify_interop.py
import asyncio
from unittest.mock import AsyncMock, MagicMock

from amplifier_module_loop_agent import AgentOrchestrator
from amplifier_module_loop_agent.config import SessionConfig

from amplifier_module_loop_pipeline.dot_parser import parse_dot
from amplifier_module_loop_pipeline.engine import PipelineEngine
from amplifier_module_loop_pipeline.context import PipelineContext
from amplifier_module_loop_pipeline.handlers import HandlerRegistry
from amplifier_module_loop_pipeline.graph import Node
from amplifier_module_loop_pipeline.outcome import Outcome, StageStatus
from amplifier_module_loop_pipeline.validation import validate_or_raise

from amplifier_core.message_models import ChatResponse, Usage


class AgentBackend:
    """CodergenBackend that spawns a loop-agent session for each node.

    This simulates the production flow where the pipeline spawns
    coding agent sessions via Amplifier's delegate mechanism.
    """

    def __init__(self):
        self.calls = []

    async def run(self, node: Node, prompt: str, context: PipelineContext) -> Outcome:
        self.calls.append(node.id)

        # Create a mock provider that returns a text response
        provider = AsyncMock()
        provider.complete = AsyncMock(return_value=ChatResponse(
            content=[{"type": "text", "text": f"Completed: {prompt}"}],
            tool_calls=None,
            usage=Usage(input_tokens=10, output_tokens=5, total_tokens=15),
        ))

        # Create a loop-agent orchestrator
        orch = AgentOrchestrator(coordinator=MagicMock(), config={})

        # Create mock hooks
        hooks = MagicMock()
        hooks.emit = AsyncMock(return_value=MagicMock(action="continue"))

        # Execute the prompt through the agent
        result = await orch.execute(
            prompt,
            MagicMock(),
            {"test": provider},
            {},  # No tools needed for this test
            hooks,
        )

        return Outcome(
            status=StageStatus.SUCCESS,
            notes=result[:200],
        )


async def main():
    dot_source = """
    digraph interop_test {
        graph [goal="Test pipeline-to-agent interop"]
        start [shape=Mdiamond]
        plan [prompt="Plan the work for: $goal"]
        implement [prompt="Execute the plan"]
        exit [shape=Msquare]
        start -> plan -> implement -> exit
    }
    """

    graph = parse_dot(dot_source)
    validate_or_raise(graph)

    import tempfile
    logs_root = tempfile.mkdtemp(prefix="interop-test-")

    backend = AgentBackend()
    context = PipelineContext()
    registry = HandlerRegistry(backend=backend)
    engine = PipelineEngine(
        graph=graph, context=context, handler_registry=registry,
        logs_root=logs_root,
    )

    outcome = await engine.run()

    assert outcome.status in (StageStatus.SUCCESS, StageStatus.PARTIAL_SUCCESS)
    assert "plan" in backend.calls
    assert "implement" in backend.calls
    print(f"Pipeline-to-agent interop: OK (status={outcome.status.value})")
    print(f"  Nodes executed via agent: {backend.calls}")

asyncio.run(main())
```

```bash
uv run python verify_interop.py
# Expected:
#   Pipeline-to-agent interop: OK (status=success)
#   Nodes executed via agent: ['plan', 'implement']
```

### Step 6: Live Provider Smoke Test (Optional, Requires API Keys)

If API keys are available, run a live end-to-end test:

```bash
# Set API keys
export ANTHROPIC_API_KEY=...
export OPENAI_API_KEY=...
export GOOGLE_API_KEY=...

# Run the integration smoke test against each provider
# (This requires a test harness that connects real providers
# to the loop-agent orchestrator — not yet implemented as a
# standalone script, but can be run through an Amplifier session)
```

This step is deferred until the modules are pushed and installable in a real Amplifier session with provider modules.

## Expected Results

| Step | What it validates | Expected outcome |
|------|-------------------|------------------|
| 1 | Shadow env creation | Environment created with local sources |
| 2 | Unit tests | 250+ loop-agent tests pass, 453+ loop-pipeline tests pass |
| 3 | Module mounting | Both modules mount as "orchestrator" |
| 4 | Import compatibility | All cross-module imports succeed |
| 5 | Pipeline-agent interop | Pipeline spawns agent sessions, executes nodes |
| 6 | Live providers | (Optional) Real LLM calls work through the agent loop |

## Interop Architecture

The key interop point is between `loop-pipeline` and `loop-agent`:

```
PipelineOrchestrator.execute()
  └── PipelineEngine.run()
        └── CodergenHandler.execute()
              └── CodergenBackend.run(node, prompt, context)
                    └── [In production: Amplifier session with loop-agent]
                          └── AgentOrchestrator.execute(prompt, ...)
                                └── AgentSession.process_input(prompt)
                                      └── provider.complete() → tool calls → loop
```

In the shadow environment, the `CodergenBackend` is a thin adapter that:
1. Creates an Amplifier session configured with the `loop-agent` module
2. Passes the node's prompt as the user input
3. Waits for the agent loop to complete
4. Returns the result as an `Outcome`

This adapter is provided by `amplifier-foundation`'s session spawning infrastructure (the `AmplifierBackend` class in `loop-pipeline/backend.py`).

## Failure Modes to Watch For

- **Import errors**: Version mismatch between `amplifier-core` models used by both modules
- **Protocol mismatches**: `ChatRequest`/`ChatResponse` fields expected by loop-agent but not provided by providers
- **Event emission failures**: Hooks protocol differences between loop-agent and loop-pipeline
- **Checkpoint serialization**: Pipeline checkpoint format incompatible with context state from agent sessions
- **Timeout propagation**: Pipeline node timeout not reaching the agent session's command timeout
