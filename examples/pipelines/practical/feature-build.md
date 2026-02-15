# Feature Build Pipeline

Parse a spec, break into subtasks, implement in parallel, integration test, human review.

## Usage

```bash
amp run --dot-file examples/pipelines/practical/feature-build.dot \
    --goal "Add user avatar upload with S3 storage and thumbnail generation"
```

## What It Does

1. **Parse Spec** -- Breaks the feature into data model, business logic, API, and test components
2. **Plan Subtasks** -- Creates 2-3 independent, non-conflicting implementation tasks
3. **Parallel Implement** -- Simultaneously builds core logic, API layer, and unit tests
4. **Integration Test** -- Runs all tests together, fixes integration issues (retry loop)
5. **Human Review** -- Pauses for human approval before finalizing (Ship or Rework)

## Key Features

- **Parallel implementation** of independent subtasks for faster builds
- **plan_subtasks** explicitly ensures no file conflicts between parallel branches
- **Human gate** before finalization gives the developer a review checkpoint
- **Integration test retry** catches cross-branch issues automatically

## Model Recommendation

Claude Sonnet for all implementation nodes (strong tool use). Optionally add a model stylesheet with o3-mini for parse_spec if you want stronger planning.
