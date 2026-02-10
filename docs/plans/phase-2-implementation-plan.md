# Phase 2 Implementation Plan

**Date:** 2026-02-08
**Source:** `phase-2-provider-improvements.md` (design document)
**Method:** TDD — failing test first, then minimal implementation, then verify

---

## Overview

7 steps, executed in order. Each step has sub-tasks organized as: test → implement → verify. Each sub-task is 2-10 minutes of work for an implementer agent.

**Repos touched per step:**
- Steps 1-6: Individual provider repos (`amplifier-module-provider-{name}/`)
- Step 7: `amplifier-module-loop-basic/`

**All provider repos share the same structure:**
- Source: `amplifier_module_provider_{name}/__init__.py`
- Tests: `tests/` directory
- Config: `pyproject.toml` with `amplifier-core` dependency

---

## Step 1: OpenAI Provider

**Repo:** `amplifier-module-provider-openai/`
**Source:** `amplifier_module_provider_openai/__init__.py` (~1,819 lines)

### Task 1.1: Error Translation

**Test** (`tests/test_error_translation.py`):
- Mock `openai.RateLimitError` raised from SDK → assert `amplifier_core.RateLimitError` raised with `provider="openai"`, `retryable=True`, `__cause__` is the native error
- Mock `openai.AuthenticationError` → assert `amplifier_core.AuthenticationError` raised with `provider="openai"`, `retryable=False`
- Mock `openai.APIStatusError` with status 500 → assert `amplifier_core.ProviderUnavailableError` raised with `retryable=True`
- Mock `asyncio.TimeoutError` → assert `amplifier_core.LLMTimeoutError` raised with `retryable=True`
- Mock generic `Exception` → assert `amplifier_core.LLMError` raised with `retryable=True` (unknown defaults to retryable)
- Mock `openai.BadRequestError` with message containing "context length" → assert `amplifier_core.ContextLengthError`

**Implement:**
- Add nested try/except inside `_complete_chat_request()` around the API call
- Inner try catches each `openai.*` error type and translates to kernel type using `raise X from e`
- Preserve existing `llm:response` error event emission

**Verify:** Run tests. Existing tests still pass.

### Task 1.2: Retry Pattern

**Test** (`tests/test_retry.py`):
- Mock provider raising `amplifier_core.RateLimitError(retryable=True)` on first 2 calls, success on 3rd → assert 3 total calls, response returned
- Mock raising `amplifier_core.RateLimitError` with `retry_after=120.0` and `max_retry_delay=60.0` → assert raises immediately (retry_after > max_delay)
- Mock raising `amplifier_core.AuthenticationError(retryable=False)` → assert raises immediately, no retry
- Assert `provider:retry` event emitted with correct `attempt`, `delay`, `error_type` fields
- Assert delay follows exponential backoff pattern (1s, 2s, 4s base)

**Implement:**
- Set `max_retries=0` on `AsyncOpenAI` client initialization (disable SDK retries)
- Add retry config reading: `max_retries`, `min_retry_delay`, `max_retry_delay`, `retry_jitter` from config dict
- Add `_calculate_retry_delay()` static method
- Wrap translated API call in retry loop (outer try catches `LLMError`, checks `retryable`)
- Emit `provider:retry` event via coordinator hooks

**Verify:** Run tests. Confirm OpenAI SDK's implicit retries are gone.

### Task 1.3: Usage Fields

**Test** (`tests/test_usage_fields.py`):
- Mock API response with `usage.output_tokens_details.reasoning_tokens=500` → assert `response.usage.reasoning_tokens == 500`
- Mock API response without `output_tokens_details` → assert `response.usage.reasoning_tokens is None`
- Existing usage fields (`input_tokens`, `output_tokens`, `total_tokens`) still populated correctly

**Implement:**
- In `_convert_to_chat_response()`, extract `reasoning_tokens` from `response.usage.output_tokens_details.reasoning_tokens` using safe `getattr` chain
- Pass to `Usage()` constructor as named field

**Verify:** Run tests.

### Task 1.4: reasoning_effort Support

**Test** (`tests/test_reasoning_effort.py`):
- Create `ChatRequest` with `reasoning_effort="high"` → assert provider passes `reasoning={"effort": "high", "summary": ...}` to API
- Create `ChatRequest` with `reasoning_effort="low"` → assert `reasoning={"effort": "low", ...}`
- Create `ChatRequest` with `reasoning_effort=None` → assert no `reasoning` param sent (existing behavior)
- Create `ChatRequest` with `reasoning_effort="medium"` AND `kwargs["reasoning"]={"effort": "high"}` → assert kwargs wins (backward compat)

**Implement:**
- In `_complete_chat_request()`, read `request.reasoning_effort`
- If set and no `kwargs["reasoning"]` override, build `reasoning` param dict
- Slot into existing reasoning parameter logic (after kwargs check, before config default)

**Verify:** Run tests.

### Task 1.5: Verify Azure OpenAI Inheritance

**Test** (in `amplifier-module-provider-azure-openai/tests/test_inherits_phase2.py`):
- Import `_AzureOpenAIProvider` (or verify via mount)
- Assert it has the retry config attributes from the updated OpenAI base
- Assert error translation works through inheritance (mock test)

**Implement:** Nothing — Azure inherits. This task is verification only.

**Verify:** Run Azure OpenAI tests.

---

## Step 2: Gemini Provider

**Repo:** `amplifier-module-provider-gemini/`
**Source:** `amplifier_module_provider_gemini/__init__.py` (~1,065 lines)

### Task 2.1: Error Translation

**Test** (`tests/test_error_translation.py`):
- Mock `google.api_core.exceptions.ResourceExhausted` → assert `amplifier_core.RateLimitError`
- Mock `google.api_core.exceptions.Unauthenticated` → assert `amplifier_core.AuthenticationError`
- Mock `google.api_core.exceptions.PermissionDenied` → assert `amplifier_core.AuthenticationError`
- Mock `google.api_core.exceptions.InvalidArgument` → assert `amplifier_core.InvalidRequestError`
- Mock `google.api_core.exceptions.ServiceUnavailable` → assert `amplifier_core.ProviderUnavailableError`
- Mock `google.api_core.exceptions.DeadlineExceeded` → assert `amplifier_core.LLMTimeoutError`
- Mock `asyncio.TimeoutError` → assert `amplifier_core.LLMTimeoutError`
- Mock generic `Exception` → assert `amplifier_core.LLMError(retryable=True)`
- All: verify `provider="gemini"` and `__cause__` preserved

**Implement:**
- Add nested try/except in `_complete_chat_request()` around the API call
- Import `google.api_core.exceptions` (may need to handle import error if not installed)
- Translate each exception type

**Verify:** Run tests.

### Task 2.2: Retry Pattern

**Test** (`tests/test_retry.py`):
- Same pattern as OpenAI Task 1.2 adapted for Gemini
- Mock `ProviderUnavailableError` on first 2 calls → success on 3rd
- Assert `provider:retry` event emitted
- Assert exponential backoff delays

**Implement:**
- Add retry config reading from config dict (same defaults as OpenAI)
- Add `_calculate_retry_delay()` method
- Wrap translated API call in retry loop

**Verify:** Run tests. This is the highest-priority fix — Gemini currently has zero retry protection.

### Task 2.3: Usage Fields

**Test** (`tests/test_usage_fields.py`):
- Mock response with `usage_metadata.thoughts_token_count=200` → assert `response.usage.reasoning_tokens == 200`
- Mock response with `usage_metadata.cached_content_token_count=1000` → assert `response.usage.cache_read_tokens == 1000`
- Mock response without these fields → assert both are `None`

**Implement:**
- In `_convert_to_chat_response()`, extract `thoughts_token_count` and `cached_content_token_count` from `response.usage_metadata` using `getattr`
- Pass to `Usage()` constructor

**Verify:** Run tests.

### Task 2.4: reasoning_effort Support

**Test** (`tests/test_reasoning_effort.py`):
- `reasoning_effort="low"` → assert `thinking_budget=4096` in config
- `reasoning_effort="medium"` → assert `thinking_budget=-1` (dynamic)
- `reasoning_effort="high"` → assert `thinking_budget=-1` (dynamic)
- `reasoning_effort=None` → existing behavior unchanged
- `kwargs["thinking_budget"]=8192` overrides `reasoning_effort="low"` → assert budget is 8192

**Implement:**
- In thinking config section, check `request.reasoning_effort` after kwargs but before defaults
- Map to budget values per design doc table

**Verify:** Run tests.

### Task 2.5: Bug Fix — _repaired_tool_ids

**Test** (`tests/test_tool_repair.py`):
- Call `_find_missing_tool_results` twice with same missing tool call → assert second call does NOT re-detect the same ID
- Assert `_repaired_tool_ids` set is maintained on the provider instance

**Implement:**
- Add `self._repaired_tool_ids: set[str] = set()` to `__init__`
- Filter already-repaired IDs in `_find_missing_tool_results`
- Match Anthropic/OpenAI pattern

**Verify:** Run tests.

---

## Step 3: Anthropic Provider

**Repo:** `amplifier-module-provider-anthropic/`
**Source:** `amplifier_module_provider_anthropic/__init__.py` (~1,838 lines)

### Task 3.1: Error Translation

**Test** (`tests/test_error_translation.py`):
- Mock `anthropic.RateLimitError` → assert `amplifier_core.RateLimitError` with `retry_after` parsed from headers
- Mock `anthropic.AuthenticationError` → assert `amplifier_core.AuthenticationError`
- Mock `anthropic.BadRequestError` with "context length" message → assert `amplifier_core.ContextLengthError`
- Mock `anthropic.BadRequestError` with "content" message → assert `amplifier_core.ContentFilterError`
- Mock `anthropic.BadRequestError` with generic message → assert `amplifier_core.InvalidRequestError`
- Mock `anthropic.APIStatusError` with status 500 → assert `amplifier_core.ProviderUnavailableError`
- Mock `asyncio.TimeoutError` → assert `amplifier_core.LLMTimeoutError`
- All: `provider="anthropic"`, `__cause__` preserved

**Implement:**
- Refactor existing `except RateLimitError` in retry loop to translate to kernel type FIRST
- Add catches for other Anthropic SDK error types
- Use nested try pattern: inner translates, outer retries

**Verify:** Run tests.

### Task 3.2: Retry Refactor

**Test** (`tests/test_retry.py`):
- Mock `amplifier_core.ProviderUnavailableError` (NOT just RateLimitError) → assert retry happens
- Mock `amplifier_core.LLMTimeoutError` → assert retry happens
- Mock `amplifier_core.AuthenticationError` → assert NO retry (retryable=False)
- Mock `RateLimitError(retry_after=120.0)` with `max_retry_delay=60.0` → assert raises immediately
- Assert event name is `provider:retry` (not `anthropic:rate_limit_retry`)
- Assert final failure raises kernel error type (not `RuntimeError`)

**Implement:**
- Refactor existing retry loop to catch `LLMError` instead of `anthropic.RateLimitError`
- Check `e.retryable` instead of hardcoded error type
- Add retry-after > max_delay check
- Change event name from `anthropic:rate_limit_retry` to `provider:retry`
- Remove `RuntimeError` wrapping — let kernel error propagate directly

**Verify:** Run tests. Existing rate limit retry behavior preserved but expanded.

### Task 3.3: Usage Named Fields

**Test** (`tests/test_usage_fields.py`):
- Mock response with `cache_read_input_tokens=500` → assert `response.usage.cache_read_tokens == 500`
- Mock response with `cache_creation_input_tokens=200` → assert `response.usage.cache_write_tokens == 200`
- Assert extras still present: `response.usage.cache_read_input_tokens == 500` (backward compat)
- Assert `response.usage.reasoning_tokens is None` (Anthropic doesn't provide separate count)

**Implement:**
- In Usage construction, add `cache_read_tokens` and `cache_write_tokens` named fields
- Keep existing extras for backward compatibility

**Verify:** Run tests.

### Task 3.4: reasoning_effort Support

**Test** (`tests/test_reasoning_effort.py`):
- `reasoning_effort="low"` → assert thinking enabled with `type="enabled"`, `budget_tokens=4096`
- `reasoning_effort="medium"` → assert thinking with `type="adaptive"` (on supported models)
- `reasoning_effort="high"` → assert thinking with `type="adaptive"`
- `reasoning_effort=None` → existing behavior unchanged
- `kwargs["extended_thinking"]=True` overrides `reasoning_effort=None` → thinking enabled
- `kwargs["extended_thinking"]=False` overrides `reasoning_effort="high"` → thinking disabled

**Implement:**
- In thinking configuration section, check `request.reasoning_effort` after kwargs but before config
- If set and no kwargs override, enable thinking with appropriate mode/budget
- Map effort levels per design doc table

**Verify:** Run tests.

---

## Step 4: vLLM Provider

**Repo:** `amplifier-module-provider-vllm/`
**Source:** `amplifier_module_provider_vllm/__init__.py` (~1,312 lines)

### Task 4.1: Error Translation

**Test** (`tests/test_error_translation.py`):
- Same test patterns as OpenAI (uses same SDK error classes)
- All: `provider="vllm"`, `__cause__` preserved

**Implement:**
- Add nested try/except pattern, same as OpenAI
- Uses same `openai.*` error classes since vLLM uses the OpenAI SDK

**Verify:** Run tests.

### Task 4.2: Retry Pattern

**Test** (`tests/test_retry.py`):
- Focus on connection/timeout errors (local server scenario)
- Mock `ProviderUnavailableError` → assert retry
- Mock `LLMTimeoutError` → assert retry
- Assert `provider:retry` event

**Implement:**
- Add retry config and `_calculate_retry_delay()`
- Wrap API call in retry loop

**Verify:** Run tests.

### Task 4.3: Usage Fields

**Test** (`tests/test_usage_fields.py`):
- Same as OpenAI: `reasoning_tokens` from `output_tokens_details`
- Verify Harmony token accounting path also works (if `_token_accounting.py` is in play)

**Implement:**
- In `_convert_to_chat_response()`, extract `reasoning_tokens` same as OpenAI

**Verify:** Run tests.

### Task 4.4: reasoning_effort Support

**Test** (`tests/test_reasoning_effort.py`):
- Same as OpenAI (vLLM already has `reasoning` config with effort levels)
- `request.reasoning_effort` feeds into existing reasoning param logic

**Implement:**
- Read `request.reasoning_effort`, map to reasoning param, same as OpenAI

**Verify:** Run tests.

---

## Step 5: Ollama Provider

**Repo:** `amplifier-module-provider-ollama/`
**Source:** `amplifier_module_provider_ollama/__init__.py` (~1,631 lines)

### Task 5.1: Error Translation

**Test** (`tests/test_error_translation.py`):
- Mock `ollama.ResponseError(status_code=401)` → assert `amplifier_core.AuthenticationError`
- Mock `ollama.ResponseError(status_code=400)` → assert `amplifier_core.InvalidRequestError`
- Mock `ollama.ResponseError(status_code=500)` → assert `amplifier_core.ProviderUnavailableError`
- Mock `ConnectionError` → assert `amplifier_core.ProviderUnavailableError`
- Mock `asyncio.TimeoutError` → assert `amplifier_core.LLMTimeoutError`
- All: `provider="ollama"`, `__cause__` preserved

**Implement:**
- Add error translation in the existing `except` blocks
- Classify `ResponseError` by `.status_code`
- Translate connection-level errors
- Keep existing `_retry_with_backoff()` for connection-level retry (it catches `ConnectionError`, `TimeoutError`, `OSError` which happen BEFORE the translation layer)

**Verify:** Run tests. Existing connection retry still works.

### Task 5.2: reasoning_effort Support

**Test** (`tests/test_reasoning_effort.py`):
- `reasoning_effort="low"` → assert `think=True` in params (Ollama thinking is binary)
- `reasoning_effort="high"` → assert `think=True`
- `reasoning_effort=None` → existing behavior (config `enable_thinking` + `thinking_effort`)
- Existing `thinking_effort` config still works alongside new field

**Implement:**
- In thinking configuration section, check `request.reasoning_effort`
- Any non-None value enables thinking (Ollama is binary on/off)
- Existing config paths continue to work

**Verify:** Run tests.

### Task 5.3: Document Usage Limitation

**Implement:**
- Add docstring/comment noting that `reasoning_tokens` is always `None` for Ollama because `eval_count` includes both reasoning and output tokens with no way to separate them
- No code change to Usage construction needed — optional fields already default to `None`

**Verify:** N/A (documentation only).

---

## Step 6: Mock Provider (Optional)

**Repo:** `amplifier-module-provider-mock/`
**Source:** `amplifier_module_provider_mock/__init__.py` (~139 lines)

### Task 6.1: Add Optional Usage Fields

**Implement:**
- Optionally add `reasoning_tokens=0`, `cache_read_tokens=0`, `cache_write_tokens=0` to canned Usage objects
- This enables downstream tests that want to verify handling of these fields

**Verify:** Existing tests pass.

---

## Step 7: loop-basic Orchestrator

**Repo:** `amplifier-module-loop-basic/`
**Source:** `amplifier_module_loop_basic/__init__.py` (~755 lines)

### Task 7.1: Set reasoning_effort on ChatRequest

**Test** (`tests/test_reasoning_effort.py`):
- Configure orchestrator with `reasoning_effort: "high"` → assert ChatRequest built with `reasoning_effort="high"`
- Configure without `reasoning_effort` → assert ChatRequest has `reasoning_effort=None`
- Verify `extended_thinking` kwarg still passed for backward compat

**Implement:**
- In ChatRequest construction (line ~221), add `reasoning_effort=self.config.get("reasoning_effort")`
- Keep existing `kwargs["extended_thinking"]` logic alongside

**Verify:** Run tests.

### Task 7.2: Enrich PROVIDER_ERROR Event

**Test** (`tests/test_error_events.py`):
- Mock provider raising `amplifier_core.RateLimitError(provider="openai", status_code=429, retryable=True)` → assert `PROVIDER_ERROR` event includes `retryable=True` and `status_code=429`
- Mock provider raising generic `Exception` → assert `PROVIDER_ERROR` event has `error.type` and `error.msg` but no `retryable` field

**Implement:**
- In the `except` block around `provider.complete()` (line ~595):
  - Add `except LLMError as e` before `except Exception as e`
  - Include `retryable` and `status_code` in the event data for LLMError
  - Keep existing `except Exception` for non-LLM errors
- Import `LLMError` from `amplifier_core`

**Verify:** Run tests. Existing error handling behavior unchanged (still emits event, still re-raises).

---

## Execution Strategy

### Per-Step PR Strategy

Each step (provider) should be a separate PR for clean review:
- **PR 1:** OpenAI + Azure OpenAI verification
- **PR 2:** Gemini (including bug fixes)
- **PR 3:** Anthropic
- **PR 4:** vLLM
- **PR 5:** Ollama
- **PR 6:** Mock (optional, can bundle with any other PR)
- **PR 7:** loop-basic orchestrator

### Dependencies

```
Step 1 (OpenAI) ──→ Step 1.5 (Azure verification)
                ──→ Step 4 (vLLM reuses patterns)

Step 2 (Gemini)     independent

Step 3 (Anthropic)  independent

Step 5 (Ollama)     independent

Step 7 (loop-basic) depends on at least one provider being done
                    (to have kernel error types flowing for the enriched event test)
```

Steps 1, 2, 3, and 5 are independent and could be parallelized.

### Testing Notes

- Each provider repo has its own `pyproject.toml` and test setup
- Run `uv run pytest tests/ -v` in each repo
- Provider tests will need to mock the native SDK (anthropic, openai, google.genai, ollama) since we don't have API keys in CI
- Use `unittest.mock.AsyncMock` for async provider client calls
- Import kernel error types from `amplifier_core.llm_errors`

### Risks

1. **Native SDK error class import paths** may differ between SDK versions. Verify actual import paths during implementation.
2. **Gemini's `google.api_core.exceptions`** may not be a direct dependency — it could come transitively via `google-genai`. Verify import works.
3. **OpenAI SDK `max_retries=0`** — verify this is the correct way to disable retries (not `max_retries=None` or similar).
4. **Ollama's `ResponseError.status_code`** — verify this field is reliably set by the Ollama SDK.
5. **vLLM's Harmony token accounting** — the `_token_accounting.py` creates SDK `ResponseUsage` objects that feed into conversion. Verify the `reasoning_tokens` extraction works for both the standard path and the Harmony path.
