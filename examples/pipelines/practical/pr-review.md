# PR Review Pipeline

Multi-dimensional pull request review with parallel analysis streams.

## Usage

```bash
amp run --dot-file examples/pipelines/practical/pr-review.dot \
    --goal "Review the changes in this PR for quality and security"
```

Or via the interactive agent:
> "Run the PR review pipeline on the current branch"

## What It Does

1. **Analyze Diff** -- Reads `git diff main...HEAD` and summarizes changes
2. **Parallel Review** -- Simultaneously checks for bugs, style issues, security vulnerabilities, and performance problems
3. **Prioritize** -- Ranks all findings by severity (must-fix -> should-fix -> consider)
4. **Generate Comments** -- Creates actionable PR review comments with file paths and suggested fixes

## Model Stylesheet

- **box nodes** (review_bugs, review_style, review_security, review_perf, generate_comments): Claude Sonnet -- strong at code reading and tool use
- **.reasoning class** (prioritize): o3-mini with high reasoning effort -- better at ranking and prioritization tasks

## Expected Behavior

- Wall-clock time: roughly the same as a single review (4 reviews run in parallel)
- Output: Markdown checklist of prioritized findings with file:line references
- The `goal_gate` on `generate_comments` ensures the pipeline won't exit without producing output
