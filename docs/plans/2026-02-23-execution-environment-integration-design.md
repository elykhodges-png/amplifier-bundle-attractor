# Execution Environment Integration Design

## Goal

Integrate the execution-environments project (env-all bundle) into attractor pipelines so that pipeline nodes can execute in isolated environments (Docker containers, SSH hosts) rather than on the local host.

## Background

The Coding Agent Loop NLSpec (Section 4) defines `execution_env` as a Session-level property. The attractor pipeline orchestrator currently runs all pipeline nodes against the local host. Users need the ability to run entire pipelines against isolated environments for reproducibility, safety, and remote execution.

This design covers use case B from the NLSpec: whole pipeline against one shared container. Use cases A (per-node environments) and C (pipeline manages environments as DOT nodes) are future enhancements.

## Key Design Decisions

1. **Compose at use time** -- env-all is NOT baked into the attractor bundle. Users compose both `attractor-pipeline` and `env-all` as separate bundles. Attractor detects env tools at runtime and uses them if present. This follows Amplifier's composition philosophy and means attractor works without execution-environments for users who don't need isolation.

2. **Explicit config** -- the orchestrator config has an optional `execution_environment` block. When present, the orchestrator creates the environment before running nodes and destroys it after. When absent, behavior is unchanged (local execution). Just composing env-all doesn't trigger environment creation.

3. **NLSpec alignment** -- per the Coding Agent Loop NLSpec Section 4, `execution_env` is a Session-level property. The pipeline doesn't know about environments. The orchestrator manages the lifecycle, and child sessions attach to the shared environment.

## Architecture Overview

The integration is a thin layer in the pipeline orchestrator that optionally manages an execution environment lifecycle. No changes to `env-all` or `execution-environments` are needed -- we consume their tools as-is.

### Data Flow

```
1. User composes: attractor-pipeline + env-all (two separate bundles)
2. User configures execution_environment in orchestrator config
3. Pipeline starts:
   a. Orchestrator detects execution_environment config
   b. Calls env_create(type="docker", ...) -> gets container_id back
   c. Stores container_id in PipelineContext
4. For each pipeline node:
   a. AmplifierBackend spawns child session
   b. Child session config includes attach_to=container_id
   c. Child session's env_create(attach_to=...) creates non-owned DockerBackend
   d. Node executes tools against the shared container
   e. Child session ends -> cleanup skips non-owned instance
5. Pipeline finishes:
   a. Orchestrator calls env_destroy on the parent's instance
   b. Container is torn down
```

### What Changes (attractor bundle only)

- `PipelineOrchestrator.execute()` -- new environment setup/teardown around the engine run
- `AmplifierBackend` -- passes `attach_to` config to child session spawn
- New config schema: `execution_environment` block in orchestrator config

### What Does NOT Change

- `env-all` bundle -- consumed as-is
- `execution-environments` repo -- no changes needed
- DOT files -- environment is not a pipeline concept per the NLSpec
- `DirectProviderBackend` -- unaffected (it doesn't spawn sessions)

## Components

### PipelineOrchestrator

The `PipelineOrchestrator.execute()` method in `modules/loop-pipeline/amplifier_module_loop_pipeline/__init__.py` currently does: parse DOT, validate, build backend, create engine, run, return result.

The change wraps the engine run with optional environment lifecycle:

```python
async def execute(self, prompt, context, providers, tools, hooks, **kwargs):
    coordinator = kwargs.get("coordinator")

    # ... existing: parse DOT, validate, build backend ...

    # NEW: Environment setup (if configured)
    env_config = self.config.get("execution_environment")
    container_id = None
    if env_config and "env_create" in tools:
        result = await tools["env_create"].execute({
            "type": env_config.get("type", "docker"),
            "name": env_config.get("name", "pipeline-workspace"),
            **env_config  # pass through image, mount_cwd, etc.
        })
        container_id = json.loads(result.output).get("container_id")
        pipeline_context.set("internal.env_container_id", container_id)
        pipeline_context.set("internal.env_type", env_config.get("type", "docker"))

    try:
        # ... existing: create engine, run ...
        outcome = await engine.run()
    finally:
        # NEW: Environment teardown
        if container_id and "env_destroy" in tools:
            await tools["env_destroy"].execute({
                "instance": env_config.get("name", "pipeline-workspace")
            })

    return result
```

Key points:

- Only triggers when both `execution_environment` config AND `env_create` tool are present (graceful when env-all not composed)
- Uses `try/finally` to ensure cleanup even if pipeline fails
- Stores `container_id` in PipelineContext so the backend can pass it to child sessions
- Pass-through of backend-specific config (`image`, `mount_cwd`, `compose_files`, etc.)

### AmplifierBackend

The `AmplifierBackend` in `modules/loop-pipeline/amplifier_module_loop_pipeline/backend.py` spawns child sessions for each pipeline node. Currently it passes node config (provider, model, prompt) but nothing about execution environments.

The change: when `container_id` is in the PipelineContext, the backend passes it to the child session's spawn config so the child can attach to the existing environment.

```python
# In AmplifierBackend.run(), when preparing spawn kwargs:
container_id = context.get("internal.env_container_id")
env_type = context.get("internal.env_type")

if container_id:
    # Add env-all tools to the child session with attach_to config
    spawn_config["tools"] = spawn_config.get("tools", []) + [{
        "module": "tools-env-all",
        "config": {
            "auto_attach": {
                "type": env_type,
                "name": "pipeline-workspace",
                "attach_to": container_id,
            }
        }
    }]
```

The child session mounts env-all with this config, which triggers an auto-attach at mount time.

Both standard tools (`tool-filesystem`, `tool-bash`, `tool-search`) and `env.*` tools are available in the child session. The agent profile's system prompt can instruct which to use. Tool name aliasing is a future enhancement.

## Config Schema and User Experience

### Step 1: Compose Both Bundles

In settings or CLI:

```yaml
bundle:
  app:
    - git+https://github.com/bkrabach/amplifier-bundle-env-all@main#subdirectory=behaviors/env-all.yaml
```

Then run with the attractor pipeline bundle.

### Step 2: Add Execution Environment Config

In the pipeline profile or bundle overlay:

```yaml
session:
  orchestrator:
    module: loop-pipeline
    config:
      dot_file: ./my-pipeline.dot
      execution_environment:
        type: docker                    # required: "docker", "ssh", or "local"
        name: pipeline-workspace        # optional, defaults to "pipeline-workspace"
        # Docker-specific (passed through to env_create):
        image: python:3.12
        mount_cwd: true
        # Or SSH-specific:
        # host: 10.0.0.5
        # username: dev
        # Or Compose-specific:
        # compose_files: ["./docker-compose.yml"]
        # compose_project: my-project
      profiles:
        anthropic: attractor-anthropic
```

### Step 3: Run Normally

Pipeline creates and destroys the environment automatically. No user action needed for environment lifecycle.

### When `execution_environment` Is Omitted

Everything works as before. Local execution, standard tools. Zero behavior change for existing users.

## Error Handling

- **env-all not composed:** If `execution_environment` is configured but `env_create` tool isn't available (env-all not composed), log a warning and fall back to local execution.
- **Environment creation fails:** Pipeline fails with a clear error before any nodes run.
- **Environment destruction fails:** Log the error in the `finally` block but don't mask the pipeline outcome.

## Scope Boundaries

- This design covers use case B (whole pipeline against one environment). Use case A (per-node environments) and C (pipeline manages environments as DOT nodes) are future enhancements.
- No changes to the execution-environments project are needed. All 4 requested features (`attach_to`, owned flag, structured return values, NLSpec metadata) are already implemented.
- DOT files are not affected. Environment configuration is deployment/runtime config, not workflow definition.
- `DirectProviderBackend` is unaffected -- it doesn't spawn sessions and doesn't need execution environments.

## Testing Strategy

1. **Unit tests for PipelineOrchestrator environment lifecycle** -- mock `env_create`/`env_destroy` tools, verify create is called before engine run and destroy is called in finally block.
2. **Unit tests for AmplifierBackend attach_to passing** -- mock spawn, verify `attach_to` config is injected into child session tools when `container_id` is in context.
3. **Integration test** -- pipeline with `execution_environment` config + mock env tools, verify the full create/attach/destroy sequence across orchestrator and backend.
4. **E2E test** -- real Docker container, simple 2-node pipeline, verify files created in the container and not on the host.

## Open Questions

1. **Auto-attach mechanism:** Should the `auto_attach` config on the `tools-env-all` module be a new feature in env-all, or should the child session explicitly call `env_create(attach_to=...)` via a session-start hook? The design assumes a config-driven auto-attach, which may need a small addition to env-all's mount function.
2. **Tool name aliasing:** Making `env_exec` available as `bash`, `env_read_file` as `read_file`, etc. -- deferred to a future phase per the execution-environments Phase 2 roadmap.
