# Bug Fix Pipeline

Systematic debugging: reproduce, diagnose, fix, write regression test, verify.

## Usage

```bash
amp run --dot-file examples/pipelines/practical/bug-fix.dot \
    --goal "Fix the NullPointerError in UserService.getProfile() when user has no avatar"
```

## What It Does

1. **Reproduce** -- Writes and runs a minimal reproduction script
2. **Diagnose** -- Analyzes root cause using o3-mini (reasoning-heavy, via model stylesheet class)
3. **Implement Fix** -- Makes the minimal code change to resolve the issue
4. **Regression Test** -- Writes a test that proves the fix works
5. **Run Tests** -- Verifies all tests pass (retries fix if not)

## Model Stylesheet

- **.reasoning class** (diagnose): o3-mini with high reasoning effort -- deep root cause analysis
- **box nodes** (all others): Default provider (Claude Sonnet recommended) -- code modification and tool use

## Key Feature: Disciplined Workflow

Forces the reproduce-first pattern. The regression test ensures the bug stays fixed. The retry loop catches cases where the fix breaks other tests.
