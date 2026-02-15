# Safe Refactoring Pipeline

Analyze code smells, plan refactoring, execute with snapshot test safety net.

## Usage

```bash
amp run --dot-file examples/pipelines/practical/refactor.dot \
    --goal "Refactor src/auth/handler.py to reduce complexity and extract helper functions"
```

## What It Does

1. **Analyze Smells** -- Identifies code smells ranked by impact
2. **Plan Refactoring** -- Creates a risk-ordered plan using o3-mini (reasoning-heavy)
3. **Snapshot Tests** -- Captures baseline test results (or writes characterization tests)
4. **Implement** -- Executes the plan with small, atomic edits
5. **Run Tests** -- Verifies no regressions against baseline (retries if failures, max 2 attempts)
6. **Diff Review** -- Uses o3-mini to verify behavior preservation

## Model Stylesheet

- **.reasoning class** (plan_refactor, diff_review): o3-mini with high reasoning effort -- planning and verification
- **box nodes** (all others): Default provider (Claude Sonnet recommended) -- code analysis and modification

## Key Feature: Snapshot Safety Net

The snapshot-first approach gives a safety net. If the refactoring breaks tests, the retry loop between `run_tests` and `implement_refactor` catches regressions immediately. The diff review confirms behavior preservation.
