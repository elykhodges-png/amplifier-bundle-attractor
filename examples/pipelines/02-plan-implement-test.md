# 02 - Plan-Implement-Test Pipeline

## What This Exercises

- **Multi-node traversal**: Three codergen nodes executed sequentially
- **Multiple codergen handlers**: Each node has a distinct prompt
- **Goal gate**: `implement` has `goal_gate=true` -- the pipeline cannot exit unless it succeeded
- **Context updates**: Each stage sets `last_stage` and `last_response` in context, visible to subsequent stages
- **Variable expansion**: `$goal` is expanded in all prompts
- **Tool use across stages**: `implement` uses `write_file`, `test` uses both `write_file` and `bash`

## Pipeline Structure

```
start --> plan --> implement --> test --> done
                  (goal_gate)
```

## Expected Behavior

1. `plan` executes first -- lists 3 steps to build the function and test file, returns SUCCESS
2. `implement` executes next -- writes `calculator.py` with type hints and docstring, returns SUCCESS (must succeed due to goal_gate)
3. `test` executes -- creates `test_calculator.py` with pytest tests (positive, negative, zero, floats), then runs `pytest test_calculator.py` via bash, returns SUCCESS
4. At `done` (exit node), the engine checks goal gates:
   - `implement` has `goal_gate=true` -- was its outcome SUCCESS? Yes -> proceed
5. Pipeline completes with SUCCESS

**Files produced on disk:**
- `calculator.py` -- the implementation (created by `implement` stage)
- `test_calculator.py` -- the pytest test file (created by `test` stage)

**If `implement` had failed:**
- The pipeline would still reach `done` via the linear path
- At the exit, goal gate check would find `implement` unsatisfied
- With no `retry_target` configured, the pipeline would fail with "Unsatisfied goal gates: ['implement']"

## How to Run

```yaml
steps:
  - agent: attractor:pipeline-runner
    instruction: "Run the plan-implement-test pipeline"
    context:
      pipeline_path: "examples/pipelines/02-plan-implement-test.dot"
```

## What to Look For

- Three stage directories in logs: `plan/`, `implement/`, `test/`
- Each has `prompt.md`, `response.md`, and `status.json`
- `checkpoint.json` shows all three nodes in `completed_nodes`
- The `implement` node's status.json shows `"outcome": "success"` (goal gate satisfied)
- `calculator.py` exists on disk with typed `add(a, b)` function
- `test_calculator.py` exists on disk with four pytest test cases
- Context contains `last_stage: "test"` after completion
- Pipeline final outcome is SUCCESS
