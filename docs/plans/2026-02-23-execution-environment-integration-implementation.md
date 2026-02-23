# Execution Environment Integration — Implementation Plan

> **Execution:** Use the subagent-driven-development workflow to implement this plan.

**Goal:** Wire optional execution environment lifecycle (create/destroy) into the pipeline orchestrator so child sessions can run inside a shared Docker container (or SSH host) instead of on the local host.

**Architecture:** The `PipelineOrchestrator.execute()` method gets a thin wrapper that calls `env_create` before `engine.run()` and `env_destroy` in a `finally` block. The `AmplifierBackend._run_with_spawn()` method reads the container ID from `PipelineContext` and injects an `attach_to` config into the child session's spawn kwargs. When `execution_environment` config is present but env tools aren't composed, we log a warning and fall back to local execution.

**Tech Stack:** Python 3.11+, pytest, pytest-asyncio (strict mode), unittest.mock

**Design doc:** `docs/plans/2026-02-23-execution-environment-integration-design.md`

**Scope (v1):** Tasks 1–4 below. **Deferred:** E2E test with real Docker, tool name aliasing, `auto_attach` in env-all mount function, per-node environments, DOT-node environment management.

---

## Task 1: Environment Lifecycle in PipelineOrchestrator

Add optional `env_create`/`env_destroy` calls around `engine.run()` in `PipelineOrchestrator.execute()`. When the config has an `execution_environment` block AND the `env_create` tool is present in the tools dict, create the environment before running the engine and destroy it in a `finally` block.

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/__init__.py` (the `PipelineOrchestrator.execute()` method, lines 378–473)
- Create: `modules/loop-pipeline/tests/test_execution_environment.py`

### Step 1: Write the failing test — env_create is called before engine runs

Create the test file. This test verifies that when `execution_environment` config is present and `env_create` tool exists, the orchestrator calls `env_create` with the right arguments.

Create `modules/loop-pipeline/tests/test_execution_environment.py`:

```python
"""Tests for execution environment lifecycle in PipelineOrchestrator.

Verifies that the orchestrator optionally creates/destroys an execution
environment around the pipeline engine run when configured.
"""

import json
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from amplifier_module_loop_pipeline import PipelineOrchestrator
from amplifier_module_loop_pipeline.context import PipelineContext
from amplifier_module_loop_pipeline.outcome import Outcome, StageStatus


# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------

def _make_mock_env_create(container_id="ctr-abc123"):
    """Create a mock env_create tool that returns a container_id."""
    tool = AsyncMock()
    tool.execute = AsyncMock(return_value=MagicMock(
        success=True,
        output=json.dumps({"container_id": container_id, "name": "pipeline-workspace"}),
    ))
    return tool


def _make_mock_env_destroy():
    """Create a mock env_destroy tool."""
    tool = AsyncMock()
    tool.execute = AsyncMock(return_value=MagicMock(
        success=True,
        output=json.dumps({"status": "destroyed"}),
    ))
    return tool


MINIMAL_DOT = """
digraph {
    start [shape=Mdiamond]
    work [prompt="Do work"]
    exit [shape=Msquare]
    start -> work -> exit
}
"""


def _make_orchestrator(execution_environment=None):
    """Create a PipelineOrchestrator with optional execution_environment config."""
    config = {"dot_source": MINIMAL_DOT}
    if execution_environment is not None:
        config["execution_environment"] = execution_environment
    return PipelineOrchestrator(config)


# ---------------------------------------------------------------------------
# Task 1: Environment lifecycle tests
# ---------------------------------------------------------------------------


@pytest.mark.asyncio
async def test_env_create_called_when_configured(tmp_path):
    """env_create is called with correct args when execution_environment is configured."""
    env_create = _make_mock_env_create()
    env_destroy = _make_mock_env_destroy()

    orchestrator = _make_orchestrator(execution_environment={
        "type": "docker",
        "name": "pipeline-workspace",
        "image": "python:3.12",
        "mount_cwd": True,
    })

    # Mock the engine run to avoid needing a real backend
    with patch(
        "amplifier_module_loop_pipeline.PipelineEngine.run",
        new_callable=AsyncMock,
        return_value=Outcome(status=StageStatus.SUCCESS, notes="done"),
    ):
        await orchestrator.execute(
            prompt="Build it",
            context=None,
            providers={},
            tools={"env_create": env_create, "env_destroy": env_destroy},
            hooks=None,
            backend=MagicMock(),  # skip backend resolution
        )

    env_create.execute.assert_called_once()
    call_args = env_create.execute.call_args[0][0]
    assert call_args["type"] == "docker"
    assert call_args["name"] == "pipeline-workspace"
    assert call_args["image"] == "python:3.12"
    assert call_args["mount_cwd"] is True
```

### Step 2: Run the test to verify it fails

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_env_create_called_when_configured -v
```

Expected: FAIL — `env_create.execute` is never called because the orchestrator doesn't know about `execution_environment` config yet.

### Step 3: Implement env_create call in PipelineOrchestrator.execute()

Edit `modules/loop-pipeline/amplifier_module_loop_pipeline/__init__.py`.

Find the `execute()` method. Currently, after step 7 (resolve backend) and before step 8 (create engine), add the environment setup. Then wrap the engine run (step 11) in a `try/finally` for teardown.

The current code (lines 421–473) looks like:

```python
        # 7. Resolve backend: explicit kwarg → auto-construct from providers
        coordinator = kwargs.get("coordinator")
        backend = kwargs.get("backend")
        if backend is None:
            backend = _build_backend(providers, tools, hooks, coordinator, self.config)

        # 8. Create engine first (handlers need its _run_from method)
        ...
        # 11. Run the engine
        outcome = await engine.run(goal=prompt or None)

        # 12. Build a meaningful summary from all completed nodes
        summary = self._build_pipeline_summary(engine, outcome)

        # 13. Return the final outcome as JSON
        result = {
            ...
        }
        return json.dumps(result)
```

Replace the section from step 7 through the end of the method with:

```python
        # 7. Resolve backend: explicit kwarg → auto-construct from providers
        coordinator = kwargs.get("coordinator")
        backend = kwargs.get("backend")
        if backend is None:
            backend = _build_backend(providers, tools, hooks, coordinator, self.config)

        # 7b. Environment setup (if configured)
        env_config = self.config.get("execution_environment")
        container_id = None
        if env_config:
            if "env_create" in tools:
                env_create_args = dict(env_config)  # copy to avoid mutating config
                # Ensure required fields have defaults
                env_create_args.setdefault("type", "docker")
                env_create_args.setdefault("name", "pipeline-workspace")
                result = await tools["env_create"].execute(env_create_args)
                parsed = json.loads(result.output)
                container_id = parsed.get("container_id")
                if container_id:
                    pipeline_context.set("internal.env_container_id", container_id)
                    pipeline_context.set(
                        "internal.env_type", env_config.get("type", "docker")
                    )
                    logger.info(
                        "Execution environment created: %s (container_id=%s)",
                        env_config.get("name", "pipeline-workspace"),
                        container_id,
                    )
                else:
                    logger.warning(
                        "env_create succeeded but returned no container_id — "
                        "falling back to local execution"
                    )
            else:
                logger.warning(
                    "execution_environment configured but env_create tool not "
                    "available (env-all bundle not composed?) — falling back "
                    "to local execution"
                )

        # 8. Create engine first (handlers need its _run_from method)
        # Use a placeholder registry, then replace after wiring
        engine = PipelineEngine(
            graph=graph,
            context=pipeline_context,
            handler_registry=HandlerRegistry(backend=backend),  # temp
            logs_root=logs_root,
            hooks=hooks,
        )

        # 9. Create subgraph runner closure that delegates to engine._run_from
        async def subgraph_runner(
            node_id: str,
            branch_context: PipelineContext,
            _graph: Any,
            _logs_root: str,
        ) -> Outcome:
            """Execute a subgraph branch via the engine."""
            return await engine._run_from(node_id, context=branch_context)

        # 10. Register handlers with the subgraph runner wired in
        registry = HandlerRegistry(
            backend=backend,
            subgraph_runner=subgraph_runner,
            hooks=hooks,
        )
        engine.handler_registry = registry

        # 11. Run the engine (with environment teardown in finally)
        try:
            outcome = await engine.run(goal=prompt or None)
        finally:
            # Environment teardown
            if container_id and "env_destroy" in tools:
                try:
                    await tools["env_destroy"].execute({
                        "instance": env_config.get("name", "pipeline-workspace"),
                    })
                    logger.info(
                        "Execution environment destroyed: %s",
                        env_config.get("name", "pipeline-workspace"),
                    )
                except Exception:
                    logger.exception(
                        "Failed to destroy execution environment %s — "
                        "container may need manual cleanup",
                        env_config.get("name", "pipeline-workspace"),
                    )

        # 12. Build a meaningful summary from all completed nodes
        summary = self._build_pipeline_summary(engine, outcome)

        # 13. Return the final outcome as JSON
        result = {
            "status": outcome.status.value,
            "notes": summary,
            "failure_reason": outcome.failure_reason,
            "nodes_completed": len(engine.completed_nodes),
            "node_statuses": {
                nid: engine.node_outcomes[nid].status.value
                for nid in engine.completed_nodes
                if nid in engine.node_outcomes
            },
        }
        return json.dumps(result)
```

### Step 4: Run the test to verify it passes

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_env_create_called_when_configured -v
```

Expected: PASS

### Step 5: Write the failing test — env_destroy is called in finally block

Add to `tests/test_execution_environment.py`:

```python
@pytest.mark.asyncio
async def test_env_destroy_called_after_engine_run(tmp_path):
    """env_destroy is called in finally block after engine.run() completes."""
    env_create = _make_mock_env_create()
    env_destroy = _make_mock_env_destroy()

    orchestrator = _make_orchestrator(execution_environment={
        "type": "docker",
        "name": "my-env",
    })

    with patch(
        "amplifier_module_loop_pipeline.PipelineEngine.run",
        new_callable=AsyncMock,
        return_value=Outcome(status=StageStatus.SUCCESS, notes="done"),
    ):
        await orchestrator.execute(
            prompt="Build it",
            context=None,
            providers={},
            tools={"env_create": env_create, "env_destroy": env_destroy},
            hooks=None,
            backend=MagicMock(),
        )

    env_destroy.execute.assert_called_once()
    call_args = env_destroy.execute.call_args[0][0]
    assert call_args["instance"] == "my-env"
```

### Step 6: Run it — should already pass

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_env_destroy_called_after_engine_run -v
```

Expected: PASS (the implementation from Step 3 already includes `finally` teardown).

### Step 7: Write the failing test — env_destroy called even when engine fails

```python
@pytest.mark.asyncio
async def test_env_destroy_called_even_on_engine_failure(tmp_path):
    """env_destroy is called in finally block even if engine.run() raises."""
    env_create = _make_mock_env_create()
    env_destroy = _make_mock_env_destroy()

    orchestrator = _make_orchestrator(execution_environment={
        "type": "docker",
        "name": "pipeline-workspace",
    })

    with patch(
        "amplifier_module_loop_pipeline.PipelineEngine.run",
        new_callable=AsyncMock,
        side_effect=RuntimeError("Engine exploded"),
    ):
        with pytest.raises(RuntimeError, match="Engine exploded"):
            await orchestrator.execute(
                prompt="Build it",
                context=None,
                providers={},
                tools={"env_create": env_create, "env_destroy": env_destroy},
                hooks=None,
                backend=MagicMock(),
            )

    # Even though the engine raised, env_destroy must have been called
    env_destroy.execute.assert_called_once()
```

### Step 8: Run it

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_env_destroy_called_even_on_engine_failure -v
```

Expected: PASS

### Step 9: Write the failing test — container_id stored in PipelineContext

```python
@pytest.mark.asyncio
async def test_container_id_stored_in_pipeline_context(tmp_path):
    """container_id from env_create is stored in PipelineContext internal keys."""
    env_create = _make_mock_env_create(container_id="ctr-xyz789")
    env_destroy = _make_mock_env_destroy()
    captured_context = {}

    orchestrator = _make_orchestrator(execution_environment={
        "type": "docker",
        "name": "pipeline-workspace",
    })

    original_run = None

    async def capturing_engine_run(self_engine, *args, **kwargs):
        """Capture the pipeline context during engine.run()."""
        captured_context["container_id"] = self_engine.context.get(
            "internal.env_container_id"
        )
        captured_context["env_type"] = self_engine.context.get("internal.env_type")
        return Outcome(status=StageStatus.SUCCESS, notes="done")

    with patch(
        "amplifier_module_loop_pipeline.PipelineEngine.run",
        capturing_engine_run,
    ):
        await orchestrator.execute(
            prompt="Build it",
            context=None,
            providers={},
            tools={"env_create": env_create, "env_destroy": env_destroy},
            hooks=None,
            backend=MagicMock(),
        )

    assert captured_context["container_id"] == "ctr-xyz789"
    assert captured_context["env_type"] == "docker"
```

### Step 10: Run it

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_container_id_stored_in_pipeline_context -v
```

Expected: PASS

### Step 11: Run all Task 1 tests together

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py -v
```

Expected: All 4 tests pass.

### Step 12: Run the full test suite to check for regressions

```bash
cd modules/loop-pipeline && uv run pytest tests/ -q --tb=short
```

Expected: All existing tests still pass.

### Step 13: Commit

```
git add modules/loop-pipeline/amplifier_module_loop_pipeline/__init__.py modules/loop-pipeline/tests/test_execution_environment.py
git commit -m "feat(loop-pipeline): add execution environment lifecycle to PipelineOrchestrator

Wire optional env_create/env_destroy calls around engine.run() in
PipelineOrchestrator.execute(). When execution_environment config is
present and env_create tool is available, creates the environment
before running and destroys it in a finally block.

Stores container_id in PipelineContext as internal.env_container_id
for downstream use by AmplifierBackend.

Design: docs/plans/2026-02-23-execution-environment-integration-design.md"
```

---

## Task 2: Attach-to Passing in AmplifierBackend

When `internal.env_container_id` is set in `PipelineContext`, the `AmplifierBackend._run_with_spawn()` method injects an `attach_to` tool config into the child session's spawn kwargs so the child session can connect to the shared container.

**Files:**
- Modify: `modules/loop-pipeline/amplifier_module_loop_pipeline/backend.py` (the `_run_with_spawn()` method, lines 196–265)
- Modify: `modules/loop-pipeline/tests/test_execution_environment.py` (add new tests)

### Step 1: Write the failing test — spawn_kwargs include env tools config

Add to `tests/test_execution_environment.py`:

```python
from amplifier_module_loop_pipeline.backend import AmplifierBackend
from amplifier_module_loop_pipeline.graph import Node


# ---------------------------------------------------------------------------
# Helpers for Task 2
# ---------------------------------------------------------------------------

class SpawnCapturingCoordinator:
    """Coordinator that captures spawn kwargs for inspection."""

    def __init__(self):
        self.spawn_called = False
        self.last_spawn_kwargs = {}
        self.session = MagicMock()
        self.config = {"agents": {}}

    def get_capability(self, name):
        if name == "session.spawn":
            return self._spawn_fn
        return None

    async def _spawn_fn(self, **kwargs):
        self.spawn_called = True
        self.last_spawn_kwargs = kwargs
        return {"output": "done", "session_id": "child-1"}


def _make_node(**kwargs):
    defaults = {"id": "implement", "prompt": "Build it"}
    defaults.update(kwargs)
    return Node(**defaults)


# ---------------------------------------------------------------------------
# Task 2: Attach-to passing tests
# ---------------------------------------------------------------------------


@pytest.mark.asyncio
async def test_backend_injects_attach_to_when_container_id_in_context():
    """When container_id is in PipelineContext, spawn kwargs include env tools config."""
    coordinator = SpawnCapturingCoordinator()
    backend = AmplifierBackend(
        coordinator=coordinator,
        profiles={"anthropic": "attractor-anthropic"},
    )

    context = PipelineContext()
    context.set("internal.env_container_id", "ctr-abc123")
    context.set("internal.env_type", "docker")

    node = _make_node(attrs={"llm_provider": "anthropic"})
    await backend.run(node, "Build it", context)

    assert coordinator.spawn_called
    kwargs = coordinator.last_spawn_kwargs

    # The spawn kwargs should include tools config with attach_to
    tools_config = kwargs.get("tools")
    assert tools_config is not None
    assert len(tools_config) == 1
    assert tools_config[0]["module"] == "tools-env-all"
    assert tools_config[0]["config"]["auto_attach"]["attach_to"] == "ctr-abc123"
    assert tools_config[0]["config"]["auto_attach"]["type"] == "docker"
    assert tools_config[0]["config"]["auto_attach"]["name"] == "pipeline-workspace"
```

### Step 2: Run the test to verify it fails

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_backend_injects_attach_to_when_container_id_in_context -v
```

Expected: FAIL — spawn kwargs don't include `tools` key.

### Step 3: Implement attach-to injection in AmplifierBackend._run_with_spawn()

Edit `modules/loop-pipeline/amplifier_module_loop_pipeline/backend.py`.

In the `_run_with_spawn()` method, find the section where `spawn_kwargs` is built (lines 219–231). After the `provider_preferences` block and before the session pool block (line 234), add the environment attach-to injection:

Find this code (around line 228):

```python
        if model:
            spawn_kwargs["provider_preferences"] = [
                _ProviderPreference(provider=provider, model=model)
            ]

        # Session pool for full fidelity (spec FID-001: thread reuse)
```

Add the env injection between those blocks:

```python
        if model:
            spawn_kwargs["provider_preferences"] = [
                _ProviderPreference(provider=provider, model=model)
            ]

        # Inject execution environment attach-to config for child session
        container_id = context.get("internal.env_container_id")
        env_type = context.get("internal.env_type")
        if container_id:
            spawn_kwargs["tools"] = spawn_kwargs.get("tools", []) + [{
                "module": "tools-env-all",
                "config": {
                    "auto_attach": {
                        "type": env_type,
                        "name": "pipeline-workspace",
                        "attach_to": container_id,
                    }
                }
            }]

        # Session pool for full fidelity (spec FID-001: thread reuse)
```

### Step 4: Run the test to verify it passes

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_backend_injects_attach_to_when_container_id_in_context -v
```

Expected: PASS

### Step 5: Write the failing test — no injection when context has no container_id

```python
@pytest.mark.asyncio
async def test_backend_no_attach_to_when_no_container_id():
    """When PipelineContext has no container_id, spawn kwargs are unchanged."""
    coordinator = SpawnCapturingCoordinator()
    backend = AmplifierBackend(
        coordinator=coordinator,
        profiles={"anthropic": "attractor-anthropic"},
    )

    context = PipelineContext()  # No env keys set

    node = _make_node(attrs={"llm_provider": "anthropic"})
    await backend.run(node, "Build it", context)

    assert coordinator.spawn_called
    kwargs = coordinator.last_spawn_kwargs
    # No tools key should be present in spawn kwargs
    assert "tools" not in kwargs
```

### Step 6: Run it

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_backend_no_attach_to_when_no_container_id -v
```

Expected: PASS (the `if container_id:` guard already handles this).

### Step 7: Run all tests so far

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py -v
```

Expected: All 6 tests pass.

### Step 8: Run the full test suite

```bash
cd modules/loop-pipeline && uv run pytest tests/ -q --tb=short
```

Expected: No regressions.

### Step 9: Commit

```
git add modules/loop-pipeline/amplifier_module_loop_pipeline/backend.py modules/loop-pipeline/tests/test_execution_environment.py
git commit -m "feat(loop-pipeline): inject attach-to config in AmplifierBackend spawn

When internal.env_container_id is set in PipelineContext, the backend
injects a tools-env-all module config with attach_to into the child
session's spawn kwargs. This lets child sessions connect to the shared
execution environment without knowing they're running in a container.

Design: docs/plans/2026-02-23-execution-environment-integration-design.md"
```

---

## Task 3: Graceful Fallback When env-all Not Composed

When `execution_environment` is configured but the `env_create` tool isn't in the tools dict (because the user didn't compose the env-all bundle), the orchestrator should log a warning and proceed with local execution — not crash.

**Files:**
- Modify: `modules/loop-pipeline/tests/test_execution_environment.py` (add new tests)
- No production code changes needed — Task 1's implementation already handles this case

### Step 1: Write the test — warning logged, no crash, pipeline runs

```python
@pytest.mark.asyncio
async def test_fallback_when_env_tools_not_composed(tmp_path, caplog):
    """Pipeline runs locally with a warning when env_create tool is missing."""
    orchestrator = _make_orchestrator(execution_environment={
        "type": "docker",
        "name": "pipeline-workspace",
    })

    # tools dict has NO env_create or env_destroy
    with patch(
        "amplifier_module_loop_pipeline.PipelineEngine.run",
        new_callable=AsyncMock,
        return_value=Outcome(status=StageStatus.SUCCESS, notes="done"),
    ):
        result = await orchestrator.execute(
            prompt="Build it",
            context=None,
            providers={},
            tools={},  # No env tools!
            hooks=None,
            backend=MagicMock(),
        )

    # Pipeline should complete successfully with local execution
    parsed = json.loads(result)
    assert parsed["status"] == "success"
    # Warning should have been logged
    assert "env_create tool not available" in caplog.text or "falling back" in caplog.text.lower()
```

### Step 2: Run it

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_fallback_when_env_tools_not_composed -v
```

Expected: PASS (Task 1's `else` branch already logs the warning and continues).

### Step 3: Write the test — no env lifecycle when config absent

```python
@pytest.mark.asyncio
async def test_no_env_lifecycle_when_config_absent(tmp_path):
    """When execution_environment is not in config, env tools are ignored."""
    env_create = _make_mock_env_create()
    env_destroy = _make_mock_env_destroy()

    # No execution_environment in config
    orchestrator = _make_orchestrator(execution_environment=None)

    with patch(
        "amplifier_module_loop_pipeline.PipelineEngine.run",
        new_callable=AsyncMock,
        return_value=Outcome(status=StageStatus.SUCCESS, notes="done"),
    ):
        result = await orchestrator.execute(
            prompt="Build it",
            context=None,
            providers={},
            tools={"env_create": env_create, "env_destroy": env_destroy},
            hooks=None,
            backend=MagicMock(),
        )

    # env tools should NOT have been called
    env_create.execute.assert_not_called()
    env_destroy.execute.assert_not_called()

    # Pipeline should still complete normally
    parsed = json.loads(result)
    assert parsed["status"] == "success"
```

### Step 4: Run it

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_no_env_lifecycle_when_config_absent -v
```

Expected: PASS

### Step 5: Write the test — env_destroy failure doesn't mask pipeline outcome

```python
@pytest.mark.asyncio
async def test_env_destroy_failure_does_not_mask_outcome(tmp_path, caplog):
    """If env_destroy fails, the pipeline outcome is still returned (not masked)."""
    env_create = _make_mock_env_create()
    env_destroy = AsyncMock()
    env_destroy.execute = AsyncMock(side_effect=RuntimeError("Destroy failed"))

    orchestrator = _make_orchestrator(execution_environment={
        "type": "docker",
        "name": "pipeline-workspace",
    })

    with patch(
        "amplifier_module_loop_pipeline.PipelineEngine.run",
        new_callable=AsyncMock,
        return_value=Outcome(status=StageStatus.SUCCESS, notes="all good"),
    ):
        result = await orchestrator.execute(
            prompt="Build it",
            context=None,
            providers={},
            tools={"env_create": env_create, "env_destroy": env_destroy},
            hooks=None,
            backend=MagicMock(),
        )

    # Pipeline outcome should be SUCCESS despite destroy failure
    parsed = json.loads(result)
    assert parsed["status"] == "success"
    # Destroy failure should be logged
    assert "failed to destroy" in caplog.text.lower() or "manual cleanup" in caplog.text.lower()
```

### Step 6: Run it

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_env_destroy_failure_does_not_mask_outcome -v
```

Expected: PASS (Task 1's `try/except` in the `finally` block handles this).

### Step 7: Run all tests

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py -v
```

Expected: All 9 tests pass.

### Step 8: Run full suite

```bash
cd modules/loop-pipeline && uv run pytest tests/ -q --tb=short
```

Expected: No regressions.

### Step 9: Commit

```
git add modules/loop-pipeline/tests/test_execution_environment.py
git commit -m "test(loop-pipeline): add fallback and edge case tests for env lifecycle

Verify graceful fallback when env-all bundle not composed, no lifecycle
when execution_environment config is absent, and env_destroy failure
doesn't mask the pipeline outcome.

Design: docs/plans/2026-02-23-execution-environment-integration-design.md"
```

---

## Task 4: Integration Test — Full Lifecycle Verification

End-to-end test that wires up both `PipelineOrchestrator` and `AmplifierBackend` together with mock env tools. Verifies the full sequence: create → context propagation → attach-to injection → destroy.

**Files:**
- Modify: `modules/loop-pipeline/tests/test_execution_environment.py` (add integration tests)

### Step 1: Write the integration test

This test creates a real `PipelineOrchestrator` with a real `AmplifierBackend` (using a mock coordinator with spawn), real `PipelineContext`, and mock env tools. It verifies the full lifecycle.

Add to `tests/test_execution_environment.py`:

```python
# ---------------------------------------------------------------------------
# Task 4: Integration test — full lifecycle
# ---------------------------------------------------------------------------


class IntegrationCoordinator:
    """Coordinator for integration testing that captures all spawn calls."""

    def __init__(self):
        self.spawn_calls = []
        self.session = MagicMock()
        self.config = {"agents": {"attractor-anthropic": {"description": "test"}}}

    def get_capability(self, name):
        if name == "session.spawn":
            return self._spawn_fn
        return None

    async def _spawn_fn(self, **kwargs):
        self.spawn_calls.append(kwargs)
        return {"output": "done", "session_id": f"child-{len(self.spawn_calls)}"}


@pytest.mark.asyncio
async def test_integration_full_env_lifecycle(tmp_path):
    """Integration: env_create -> child sessions get attach_to -> env_destroy.

    Wires up PipelineOrchestrator + AmplifierBackend with mock env tools.
    Uses a simple 2-node pipeline (plan -> implement) to verify:
    1. env_create is called first with correct config
    2. Both child session spawns include attach_to in their tools config
    3. env_destroy is called last with the right instance name
    """
    env_create = _make_mock_env_create(container_id="ctr-integration-test")
    env_destroy = _make_mock_env_destroy()
    coordinator = IntegrationCoordinator()

    orchestrator = PipelineOrchestrator({
        "dot_source": """
            digraph {
                start [shape=Mdiamond]
                plan [prompt="Plan the work"]
                implement [prompt="Build it"]
                exit [shape=Msquare]
                start -> plan -> implement -> exit
            }
        """,
        "execution_environment": {
            "type": "docker",
            "name": "integration-env",
            "image": "python:3.12",
        },
        "profiles": {
            "anthropic": "attractor-anthropic",
        },
    })

    await orchestrator.execute(
        prompt="Build a feature",
        context=None,
        providers={},
        tools={"env_create": env_create, "env_destroy": env_destroy},
        hooks=None,
        coordinator=coordinator,
    )

    # 1. env_create was called once with the right config
    env_create.execute.assert_called_once()
    create_args = env_create.execute.call_args[0][0]
    assert create_args["type"] == "docker"
    assert create_args["name"] == "integration-env"
    assert create_args["image"] == "python:3.12"

    # 2. Both child sessions received attach_to config
    assert len(coordinator.spawn_calls) == 2  # plan + implement
    for spawn_call in coordinator.spawn_calls:
        tools_config = spawn_call.get("tools")
        assert tools_config is not None, "Child spawn missing tools config"
        assert len(tools_config) == 1
        auto_attach = tools_config[0]["config"]["auto_attach"]
        assert auto_attach["attach_to"] == "ctr-integration-test"
        assert auto_attach["type"] == "docker"
        assert auto_attach["name"] == "pipeline-workspace"

    # 3. env_destroy was called once with the right instance name
    env_destroy.execute.assert_called_once()
    destroy_args = env_destroy.execute.call_args[0][0]
    assert destroy_args["instance"] == "integration-env"


@pytest.mark.asyncio
async def test_integration_no_env_config_unchanged_behavior(tmp_path):
    """Integration: without execution_environment, pipeline runs as before.

    No env tools called, child sessions spawned without attach_to.
    """
    env_create = _make_mock_env_create()
    env_destroy = _make_mock_env_destroy()
    coordinator = IntegrationCoordinator()

    orchestrator = PipelineOrchestrator({
        "dot_source": """
            digraph {
                start [shape=Mdiamond]
                work [prompt="Do work"]
                exit [shape=Msquare]
                start -> work -> exit
            }
        """,
        # NO execution_environment config
        "profiles": {
            "anthropic": "attractor-anthropic",
        },
    })

    await orchestrator.execute(
        prompt="Do work",
        context=None,
        providers={},
        tools={"env_create": env_create, "env_destroy": env_destroy},
        hooks=None,
        coordinator=coordinator,
    )

    # env tools should NOT have been called
    env_create.execute.assert_not_called()
    env_destroy.execute.assert_not_called()

    # Child sessions should have been spawned WITHOUT tools config
    assert len(coordinator.spawn_calls) == 1  # work
    assert "tools" not in coordinator.spawn_calls[0]
```

### Step 2: Run the integration tests

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py::test_integration_full_env_lifecycle tests/test_execution_environment.py::test_integration_no_env_config_unchanged_behavior -v
```

Expected: Both PASS.

### Step 3: Run all tests in the file

```bash
cd modules/loop-pipeline && uv run pytest tests/test_execution_environment.py -v
```

Expected: All 11 tests pass.

### Step 4: Run the full test suite

```bash
cd modules/loop-pipeline && uv run pytest tests/ -q --tb=short
```

Expected: No regressions in any existing test file.

### Step 5: Commit

```
git add modules/loop-pipeline/tests/test_execution_environment.py
git commit -m "test(loop-pipeline): add integration test for full env lifecycle

Integration test wires up PipelineOrchestrator + AmplifierBackend with
mock env tools and a 2-node pipeline. Verifies the full sequence:
env_create -> context propagation -> attach_to injection in child
session spawns -> env_destroy.

Also verifies that without execution_environment config, existing
behavior is completely unchanged (no env tools called, no attach_to
in spawn kwargs).

Design: docs/plans/2026-02-23-execution-environment-integration-design.md"
```

---

## Final Verification

After all 4 tasks are complete, run the full suite one final time:

```bash
cd modules/loop-pipeline && uv run pytest tests/ -v
```

### Summary of Changes

| File | Change |
|------|--------|
| `amplifier_module_loop_pipeline/__init__.py` | Added env setup/teardown in `PipelineOrchestrator.execute()` |
| `amplifier_module_loop_pipeline/backend.py` | Added attach-to injection in `AmplifierBackend._run_with_spawn()` |
| `tests/test_execution_environment.py` | New file: 11 tests covering lifecycle, fallback, edge cases, integration |

### What's NOT in This Plan (Deferred)

- E2E test with real Docker (needs env-all installed + Docker daemon running)
- Tool name aliasing (`env_exec` → `bash`, `env_read_file` → `read_file`)
- `auto_attach` config in env-all's mount function
- Per-node environments (use case A from the NLSpec)
- DOT-node environment management (use case C from the NLSpec)
