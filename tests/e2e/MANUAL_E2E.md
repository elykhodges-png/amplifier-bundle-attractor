# Manual E2E Tests

End-to-end tests for the Attractor bundle. These require a real LLM provider
(Anthropic API key) and are run manually in a shadow environment or local dev setup.

## Prerequisites

- `amplifier` CLI installed and on PATH
- `ANTHROPIC_API_KEY` set in environment
- Working directory with write access (tests create files)

## Agent Tests (E2E 1-3)

Use profile: `profiles/attractor-e2e-anthropic.yaml`

These tests exercise the single-turn agent loop (`loop-agent`) with real tool calls.

### E2E Test 1: Agent creates a file

```bash
mkdir -p /tmp/e2e-test1 && cd /tmp/e2e-test1
amplifier run -B "file://<BUNDLE_ROOT>/profiles/attractor-e2e-anthropic.yaml" \
  --mode single \
  "Create a file called hello.py that prints Hello World. Use the write_file tool."
```

**Expected result:**
- `hello.py` exists in the working directory
- Contains `print` statement with "Hello World" (or similar)
- Verify: `test -f hello.py && grep -qi hello hello.py`

### E2E Test 2: Agent reads and edits a file

```bash
mkdir -p /tmp/e2e-test2 && cd /tmp/e2e-test2
echo "print('original content')" > existing.py
amplifier run -B "file://<BUNDLE_ROOT>/profiles/attractor-e2e-anthropic.yaml" \
  --mode single \
  "Read the file existing.py, then edit it to also print 'added line'. Use read_file then edit_file."
```

**Expected result:**
- `existing.py` contains both "original" and "added" text
- Verify: `grep -q 'original' existing.py && grep -q 'added' existing.py`

### E2E Test 3: Agent runs a shell command

```bash
mkdir -p /tmp/e2e-test3 && cd /tmp/e2e-test3
amplifier run -B "file://<BUNDLE_ROOT>/profiles/attractor-e2e-anthropic.yaml" \
  --mode single \
  "Run the command 'echo hello_from_shell' using the bash tool and tell me the output." \
  2>&1 | tee output.log
```

**Expected result:**
- Agent invokes the bash/shell tool
- Output log contains "hello_from_shell"
- Verify: `grep -q 'hello_from_shell' output.log`

## Pipeline Tests (E2E 4-6)

Use profile: `profiles/attractor-e2e-pipeline-anthropic.yaml`

These tests exercise the DOT graph-driven pipeline (`loop-pipeline`). Each test
uses a different DOT fixture. Pipeline tests take longer (2-5 minutes) since they
make multiple sequential LLM calls.

**Note:** The pipeline profile defaults to `simple_file_creation.dot`. For tests 5
and 6, override the `dot_file` config or create separate profile copies.

### E2E Test 4: Simple pipeline (single node)

DOT fixture: `tests/e2e/fixtures/simple_file_creation.dot`

```bash
mkdir -p /tmp/e2e-test4 && cd /tmp/e2e-test4
amplifier run -B "file://<BUNDLE_ROOT>/profiles/attractor-e2e-pipeline-anthropic.yaml" \
  --mode single \
  "Run the pipeline"
```

**Expected result:**
- Pipeline executes: start -> implement -> done
- `hello.py` is created (the implement node's prompt asks for it)
- Agent session spawned for the `implement` node completes successfully
- Verify: `test -f hello.py && python3 hello.py | grep -qi hello`

### E2E Test 5: Multi-stage pipeline (plan/implement/review)

DOT fixture: `tests/e2e/fixtures/plan_implement_review.dot`

Requires modifying the pipeline profile's `dot_file` to point to this fixture,
or passing config override.

```
Pipeline graph: start -> plan -> implement -> validate -> done
```

**Expected result:**
- All 4 nodes execute in sequence
- `plan` node produces a brief plan (visible in agent output)
- `implement` node creates `test_math.py` with `add(a, b)` function
- `validate` node runs `python3 test_math.py` and reports pass/fail
- Verify: `test -f test_math.py && python3 test_math.py`

### E2E Test 6: Conditional routing pipeline

DOT fixture: `tests/e2e/fixtures/conditional_routing.dot`

Requires modifying the pipeline profile's `dot_file` to point to this fixture.

```
Pipeline graph: start -> implement -> test -> gate
  gate -> done         [condition="outcome=success"]
  gate -> implement    [condition="outcome!=success", label="Retry"]
```

**Expected result:**
- `implement` node creates `calc.py` with `multiply(a, b)` function
- `test` node runs `python3 calc.py` and checks for "PASS" output
- `gate` node routes to `done` on success (or retries `implement` on failure)
- Pipeline terminates at `done`
- Verify: `test -f calc.py && python3 calc.py | grep -q PASS`

## Timeout Guidance

| Test Type | Typical Duration | Suggested Timeout |
|-----------|-----------------|-------------------|
| Agent (E2E 1-3) | 30-90s | 120s |
| Pipeline single-node (E2E 4) | 60-120s | 300s |
| Pipeline multi-stage (E2E 5-6) | 120-300s | 600s |

## Troubleshooting

- **"No provider configured"**: Check `ANTHROPIC_API_KEY` is set
- **Tool call failures**: Ensure the working directory is writable
- **Pipeline hangs**: Check that `dot_file` path resolves correctly relative to the profile
- **Timeout**: Pipeline tests make multiple LLM calls; increase timeout or check network
