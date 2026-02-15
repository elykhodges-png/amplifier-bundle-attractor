# Test Generation Pipeline

Generate tests, run them, and fix failures in a self-healing retry loop.

## Usage

```bash
amp run --dot-file examples/pipelines/practical/test-gen.dot \
    --goal "Generate comprehensive tests for the user authentication module in src/auth/"
```

## What It Does

1. **Analyze Module** -- Reads source files, identifies public API surface and edge cases
2. **Identify Gaps** -- Compares existing tests against the API surface
3. **Write Tests** -- Generates pytest tests covering identified gaps
4. **Run Tests** -- Executes the test suite and reports results
5. **Fix Failures** -- Diagnoses and fixes test failures (retry loop)

## Key Feature: Self-Healing Loop

The retry loop between `run_tests` and `fix_failures` means the pipeline doesn't just generate tests -- it validates them and fixes failures automatically. Up to 3 retry cycles.

## Model Recommendation

Claude Sonnet for all nodes (strong at code generation and tool use). No model stylesheet needed -- the default provider works well for all stages.
