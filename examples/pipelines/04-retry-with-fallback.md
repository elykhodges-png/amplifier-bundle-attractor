# 04 - Retry with Fallback Pipeline

## What This Exercises

- **`max_retries` attribute**: `implement` has `max_retries=2` meaning up to 3 total executions (1 initial + 2 retries)
- **`retry_target`**: When `implement` exhausts retries, jump back to `plan` for a fresh approach
- **`fallback_retry_target`**: If `retry_target` also fails or is invalid, fall back to `simple_implement`
- **Graph-level `fallback_retry_target`**: The graph itself has `fallback_retry_target="simple_implement"` as a last resort
- **Graph-level `default_max_retry`**: Sets the global retry ceiling to 3
- **`goal_gate` + retry interaction**: Both `implement` and `simple_implement` are goal gates -- the pipeline cannot exit until at least one succeeds
- **`allow_partial`**: `simple_implement` accepts PARTIAL_SUCCESS when retries exhaust, treating it as good enough

## Pipeline Structure

```
start --> plan --> implement --> validate --> done
           ^        |                |
           |        | (retry_target) |
           +--------+               |
                                     | (fail edge)
                                     +-----------> implement
           
simple_implement --> validate
(fallback_retry_target from implement and graph)
```

## Expected Behavior

### Happy Path
1. `plan` succeeds
2. `implement` succeeds on first try (goal gate satisfied)
3. `validate` succeeds
4. At `done`, goal gate check passes -> SUCCESS

### Retry Path
1. `plan` succeeds
2. `implement` fails on first try
3. Engine retries `implement` (max_retries=2, so up to 2 more attempts)
4. If `implement` succeeds on retry -> continue to `validate`
5. If all 3 attempts fail -> jump to `retry_target="plan"` for a fresh plan

### Fallback Path
1. After `retry_target="plan"` is exhausted:
2. Engine falls back to `fallback_retry_target="simple_implement"`
3. `simple_implement` runs with `allow_partial=true`
4. Even PARTIAL_SUCCESS satisfies the goal gate
5. Continues to `validate` -> `done`

### Goal Gate Enforcement at Exit
When reaching `done`, the engine checks:
- Was `implement`'s outcome SUCCESS or PARTIAL_SUCCESS? 
- Or was `simple_implement`'s outcome SUCCESS or PARTIAL_SUCCESS?
- If neither goal gate is satisfied, the engine uses the retry target chain

## How to Run

```yaml
steps:
  - agent: attractor:pipeline-runner
    instruction: "Run the retry with fallback pipeline"
    context:
      pipeline_path: "examples/pipelines/04-retry-with-fallback.dot"
```

## What to Look For

- Check `implement/status.json` for retry count in the checkpoint's `node_retries`
- If fallback activated: `simple_implement/` directory appears in logs
- `checkpoint.json` shows which nodes were completed and their outcomes
- Goal gate check: look for `pipeline:goal_gate_check` events showing satisfied/unsatisfied lists
- `allow_partial` behavior: if `simple_implement` returns PARTIAL_SUCCESS, it still satisfies the gate
