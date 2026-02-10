# Phase 2 Design: Provider Improvements

**Date:** 2026-02-08
**Status:** Design Complete — Ready for Review
**Prerequisite:** Phase 1 merged (amplifier-core PR #10)
**Scope:** All 7 official provider modules + minimal orchestrator touchpoints

---

## Context

Phase 1 added kernel vocabulary to amplifier-core:
- `Usage` with `reasoning_tokens`, `cache_read_tokens`, `cache_write_tokens`
- `ChatRequest` with `model`, `tool_choice`, `stop`, `reasoning_effort`, `timeout`
- Error taxonomy: `LLMError`, `RateLimitError`, `AuthenticationError`, `ContextLengthError`, `ContentFilterError`, `InvalidRequestError`, `ProviderUnavailableError`, `LLMTimeoutError`

Phase 2 makes all provider modules the first consumers of this vocabulary. The goal is: after Phase 2, any hook on `provider:response` can read `usage.reasoning_tokens` from any provider, any hook can catch `RateLimitError` regardless of provider, and transient errors are retried consistently.

---

## The Full Provider Landscape

| Provider | Size | Based On | Phase 2 Tier |
|---|---|---|---|
| **Anthropic** | 1,838 lines | Native Anthropic SDK | Tier 1: Full treatment |
| **OpenAI** | 1,819 lines | Native Responses API | Tier 1: Full treatment |
| **Gemini** | 1,065 lines | Native GenAI SDK | Tier 1: Full treatment |
| **Ollama** | 1,631 lines | Native ollama SDK | Tier 3: Adapt to existing patterns |
| **vLLM** | ~1,977 lines | OpenAI SDK (standalone copy) | Tier 2: Reuse OpenAI patterns |
| **Azure OpenAI** | 401 lines | Inherits from OpenAI at runtime | Tier 2: Inherits from OpenAI |
| **Mock** | 139 lines | None | Tier 4: Optional (testing only) |

### Tier Definitions

**Tier 1 — Full treatment:** Complete error translation, retry pattern, usage field mapping, reasoning_effort support. These are the reference implementations that the Attractor nlspec targets.

**Tier 2 — Follows from Tier 1:** Shares code or inherits from a Tier 1 provider. Changes flow through automatically or with minimal adaptation.
- **Azure OpenAI** dynamically inherits from `OpenAIProvider` at runtime — all Phase 2 changes to OpenAI apply automatically. Verify Azure-specific error codes only.
- **vLLM** is a standalone copy of the OpenAI provider's patterns (same SDK, same Responses API). Reuse OpenAI's error mapping and retry pattern with vLLM-specific adaptations.

**Tier 3 — Adapt to existing patterns:** Independent provider with its own established patterns. Phase 2 aligns terminology and adds error taxonomy, but respects the provider's existing design decisions.
- **Ollama** already has its own retry logic, thinking support, and unique Usage field names. Phase 2 adds error translation and standardizes naming, but keeps Ollama's existing retry for connection-level errors.

**Tier 4 — Optional:** Testing infrastructure. Optionally surface new fields for test coverage.

---

## Design Decisions

### 1. Retry: Provider-Internal, Following a Shared Pattern

**Decision:** Each provider implements retry internally, following a documented pattern. No shared utility package.

**Rationale:**
- Amplifier's module system doesn't have a "shared utilities" layer. Modules are independent bricks. Creating a shared dependency between provider modules violates the brick philosophy.
- The retry loop is ~30 lines. Duplicating across providers is less coupling overhead than a shared import.
- The kernel error taxonomy already provides the classification vocabulary — providers translate native errors to kernel types, then use `error.retryable` to decide.
- The provider has native error context (headers, status codes, error body) that a shared utility would need passed in anyway.

**The Pattern (cloud providers):**

```python
last_error: LLMError | None = None

for attempt in range(1, max_retries + 2):
    try:
        response = await _call_api(params)
        break
    except LLMError as e:
        if not e.retryable or attempt > max_retries:
            raise
        last_error = e
        retry_after = getattr(e, "retry_after", None)
        if retry_after is not None and retry_after > max_retry_delay:
            raise  # Don't silently wait minutes — fail fast
        delay = _calculate_delay(retry_after, attempt, min_delay, max_delay, jitter)
        await hooks.emit("provider:retry", {
            "provider": self.name, "attempt": attempt,
            "delay": delay, "error_type": type(e).__name__,
        })
        await asyncio.sleep(delay)
```

**Delay calculation** (same across providers):
```python
def _calculate_delay(retry_after, attempt, min_delay, max_delay, jitter):
    if retry_after is not None and retry_after > 0:
        delay = retry_after
    else:
        delay = min_delay * (2 ** (attempt - 1))
    delay = min(delay, max_delay)
    if jitter:
        delay *= random.uniform(0.8, 1.2)
    return delay
```

**What changes from Anthropic's current implementation:**
- Catches `LLMError` (any retryable kernel error) instead of only `anthropic.RateLimitError`
- Also retries on `ProviderUnavailableError` and `LLMTimeoutError` (both `retryable=True`)
- Honors the "retry-after exceeds max_delay = raise immediately" rule (Anthropic currently caps and retries anyway)
- Normalizes event name to `provider:retry` (not `anthropic:rate_limit_retry`)
- Final failure raises the kernel error type, not `RuntimeError`

**Config surface** (same for all providers that implement retry):
```yaml
max_retries: 5        # default, existing Anthropic value
min_retry_delay: 1.0  # seconds
max_retry_delay: 60.0 # seconds
retry_jitter: true
```

#### Per-Provider Retry Strategy

| Provider | Current State | Phase 2 Change |
|---|---|---|
| **Anthropic** | Hand-rolled retry for `RateLimitError` only (5 retries, backoff, jitter) | Refactor to use kernel errors. Expand to retry all `retryable=True` errors. |
| **OpenAI** | Invisible SDK default (2 retries) | Disable SDK retries (`max_retries=0`). Add Phase 2 pattern for observable, configurable retry. |
| **Gemini** | Zero retry | Add Phase 2 pattern. Highest priority fix. |
| **vLLM** | Zero retry | Add Phase 2 pattern. Primarily retries connection/timeout errors (local server). |
| **Azure OpenAI** | Inherits from OpenAI | Inherits the fix. Verify Azure-specific transient errors are covered. |
| **Ollama** | Has own `_retry_with_backoff()` for `ConnectionError`, `TimeoutError`, `OSError` (3 attempts) | Keep existing retry for connection-level errors. Add error translation so kernel types flow. Ollama's retry is appropriate for local servers — no rate limits to handle. |
| **Mock** | N/A | N/A |

#### Local vs Cloud Retry Distinction

The Phase 2 retry pattern works the same everywhere, but the error landscape differs:

- **Cloud providers** (Anthropic, OpenAI, Gemini, Azure) raise `RateLimitError`, `ProviderUnavailableError`, `LLMTimeoutError` — all retried.
- **Local providers** (Ollama, vLLM) never raise `RateLimitError` — they primarily see `ProviderUnavailableError` (server restart) and `LLMTimeoutError` (slow inference). The same retry pattern handles this naturally because it keys on `error.retryable`, not error type.
- **Ollama** additionally retries OS-level connection errors (`ConnectionError`, `OSError`) that aren't LLM errors. Its existing `_retry_with_backoff()` handles this layer. Phase 2 adds the kernel error translation layer on top without replacing the connection retry.

---

### 2. Error Translation: Nested Try Pattern

**Decision:** Translate native SDK errors to kernel types inside each provider's existing `except` blocks. Use `raise X from native_error` to preserve the chain. Separate translation from retry with a nested try structure.

**Rationale:**
- Each provider already has try/except structure around the API call
- Minimal structural change — we're replacing `raise` with `raise KernelType(...) from e`
- Provider-specific context (model, params) is available at the catch site
- `except Exception` in orchestrators continues to work (kernel errors are Exceptions)

**Implementation structure:**

```python
for attempt in range(1, max_retries + 2):
    try:
        try:
            response = await _call_api(params)
            break
        except native_sdk.RateLimitError as e:
            raise kernel_RateLimitError(..., retry_after=parsed) from e
        except native_sdk.AuthenticationError as e:
            raise kernel_AuthenticationError(...) from e
        except native_sdk.BadRequestError as e:
            raise _classify_bad_request(e) from e
        except native_sdk.APIStatusError as e:
            raise _classify_by_status(e) from e
        except asyncio.TimeoutError as e:
            raise kernel_LLMTimeoutError(...) from e
        except Exception as e:
            raise LLMError(str(e), provider=self.name, retryable=True) from e
    except LLMError as e:
        if not e.retryable or attempt > max_retries:
            raise
        # ... retry logic
```

The inner try translates. The outer try retries. Clean separation.

**Unknown errors default to `retryable=True`** per the Attractor spec's principle that transient failures are more common than permanent ones.

#### Per-Provider Error Mapping

##### Anthropic
| SDK Exception | Kernel Type | Notes |
|---|---|---|
| `anthropic.RateLimitError` | `RateLimitError` | Parse retry-after from headers |
| `anthropic.AuthenticationError` | `AuthenticationError` | |
| `anthropic.BadRequestError` | Classify by message | Context length, content filter, or invalid request |
| `anthropic.APIStatusError` (5xx) | `ProviderUnavailableError` | |
| `asyncio.TimeoutError` | `LLMTimeoutError` | |
| Other `Exception` | `LLMError(retryable=True)` | Unknown defaults to retryable |

**Anthropic message classification** (for `BadRequestError`):
```python
msg = str(e).lower()
if "context length" in msg or "too many tokens" in msg:
    raise ContextLengthError(...) from e
elif "content filter" in msg or "safety" in msg or "blocked" in msg:
    raise ContentFilterError(...) from e
else:
    raise InvalidRequestError(...) from e
```

##### OpenAI
| SDK Exception | Kernel Type |
|---|---|
| `openai.RateLimitError` | `RateLimitError` |
| `openai.AuthenticationError` | `AuthenticationError` |
| `openai.BadRequestError` | Classify by message (same approach) |
| `openai.APIStatusError` (5xx) | `ProviderUnavailableError` |
| `asyncio.TimeoutError` | `LLMTimeoutError` |
| Other `Exception` | `LLMError(retryable=True)` |

##### Gemini
| SDK Exception | Kernel Type |
|---|---|
| `google.api_core.exceptions.ResourceExhausted` | `RateLimitError` |
| `google.api_core.exceptions.Unauthenticated` | `AuthenticationError` |
| `google.api_core.exceptions.PermissionDenied` | `AuthenticationError` |
| `google.api_core.exceptions.InvalidArgument` | `InvalidRequestError` |
| `google.api_core.exceptions.ServiceUnavailable` | `ProviderUnavailableError` |
| `google.api_core.exceptions.DeadlineExceeded` | `LLMTimeoutError` |
| `asyncio.TimeoutError` | `LLMTimeoutError` |
| Other `Exception` | `LLMError(retryable=True)` |

##### vLLM
Uses the OpenAI SDK — same error mapping as OpenAI. The OpenAI SDK's error classes (`openai.RateLimitError`, etc.) are the same regardless of what backend the SDK points at.

##### Azure OpenAI
Inherits from OpenAI. Additional consideration: Azure returns specific content filter error codes that standard OpenAI doesn't. These should map to `ContentFilterError`. Azure managed identity token refresh failures should map to `AuthenticationError`.

##### Ollama
| SDK Exception | Kernel Type | Notes |
|---|---|---|
| `ollama.ResponseError` (status 401/403) | `AuthenticationError` | Rare for local, but possible with remote Ollama |
| `ollama.ResponseError` (status 400) | `InvalidRequestError` | Model not found, bad params |
| `ollama.ResponseError` (status 5xx) | `ProviderUnavailableError` | Server internal error |
| `ConnectionError` | `ProviderUnavailableError` | Server not running |
| `OSError` | `ProviderUnavailableError` | Network-level failure |
| `asyncio.TimeoutError` | `LLMTimeoutError` | 600s default for local models |
| Other `Exception` | `LLMError(retryable=True)` | |

Note: Ollama's `ResponseError` has a `.status_code` field that enables classification. No rate limits for local servers.

##### Mock
No error translation needed — the mock doesn't make network calls.

---

### 3. Usage Fields: Set Both Named Fields and Provider-Native Extras

**Decision:** Set the new named kernel fields AND continue passing provider-native extras for backward compatibility.

#### Anthropic
```python
usage = Usage(
    input_tokens=response.usage.input_tokens,
    output_tokens=response.usage.output_tokens,
    total_tokens=input + output,
    # NEW named fields
    cache_read_tokens=getattr(response.usage, "cache_read_input_tokens", None) or None,
    cache_write_tokens=getattr(response.usage, "cache_creation_input_tokens", None) or None,
    # KEEP extras for backward compat
    cache_creation_input_tokens=getattr(response.usage, "cache_creation_input_tokens", None),
    cache_read_input_tokens=getattr(response.usage, "cache_read_input_tokens", None),
)
```

**Note on Anthropic `reasoning_tokens`:** Anthropic does not provide a separate reasoning token count. Thinking tokens are included in `output_tokens`. We will NOT estimate from thinking block text length — the kernel field should report what the provider actually tells us. `reasoning_tokens` stays `None` for Anthropic until they add a dedicated field.

#### OpenAI
```python
reasoning_tokens = None
if usage_obj and hasattr(usage_obj, "output_tokens_details"):
    details = usage_obj.output_tokens_details
    if details and hasattr(details, "reasoning_tokens"):
        reasoning_tokens = details.reasoning_tokens

usage = Usage(
    input_tokens=...,
    output_tokens=...,
    total_tokens=...,
    reasoning_tokens=reasoning_tokens,
)
```

The Responses API returns `output_tokens_details.reasoning_tokens` for reasoning-capable models. For non-reasoning models, this field is absent — `reasoning_tokens` stays `None`.

#### Gemini
```python
thoughts_tokens = getattr(response.usage_metadata, "thoughts_token_count", None) or None
cached_tokens = getattr(response.usage_metadata, "cached_content_token_count", None) or None

usage = Usage(
    input_tokens=prompt_token_count,
    output_tokens=candidates_token_count,
    total_tokens=total_token_count,
    reasoning_tokens=thoughts_tokens,
    cache_read_tokens=cached_tokens,
)
```

Both fields exist in Gemini's `usage_metadata` today but are currently ignored.

#### vLLM
Same as OpenAI for the standard path. The Harmony token accounting path (`_token_accounting.py`) creates OpenAI SDK `ResponseUsage` objects — these feed into the same conversion, so the `reasoning_tokens` extraction works identically.

#### Azure OpenAI
Inherited from OpenAI — no changes needed.

#### Ollama
```python
usage = Usage(
    input_tokens=response.get("prompt_eval_count", 0),
    output_tokens=response.get("eval_count", 0),
    total_tokens=prompt_eval_count + eval_count,
    # reasoning_tokens: None — Ollama lumps thinking tokens into eval_count
    # cache_read_tokens: None — not applicable for local models
    # cache_write_tokens: None — not applicable for local models
)
```

**Ollama limitation:** Ollama's `eval_count` includes both reasoning and output tokens with no way to separate them. `reasoning_tokens` must always be `None`. This is a data accuracy limitation to document, not a code gap.

#### Mock
Optionally add `reasoning_tokens=0`, `cache_read_tokens=0`, `cache_write_tokens=0` to canned responses for testing downstream handling. Not required.

#### Usage Field Availability Matrix

| Field | Anthropic | OpenAI | Gemini | vLLM | Azure | Ollama | Mock |
|---|---|---|---|---|---|---|---|
| `input_tokens` | Yes | Yes | Yes | Yes | Yes | Yes | Canned |
| `output_tokens` | Yes | Yes | Yes | Yes | Yes | Yes | Canned |
| `total_tokens` | Computed | Computed | From API | Computed | Inherited | Computed | Canned |
| `reasoning_tokens` | `None` | From API | From API | From API | Inherited | `None` | Optional |
| `cache_read_tokens` | From API | `None` | From API | `None` | Inherited | `None` | Optional |
| `cache_write_tokens` | From API | `None` | `None` | `None` | Inherited | `None` | Optional |

---

### 4. Thinking Trigger: `reasoning_effort` Checked First, kwargs Override

**Decision:** Providers check `request.reasoning_effort` as the portable interface, with kwargs providing backward-compatible override.

**Precedence chain:**
```
kwargs["extended_thinking"] or kwargs["reasoning"]  ->  overrides everything (backward compat)
request.reasoning_effort                            ->  portable, new way
config default                                      ->  session-level default
None                                                ->  no opinion, existing behavior
```

#### Per-Provider Mapping

##### OpenAI
Nearly 1:1 — `reasoning_effort` maps directly to `reasoning.effort`:

| `reasoning_effort` | OpenAI `reasoning` param |
|---|---|
| `"low"` | `{"effort": "low", "summary": config.reasoning_summary}` |
| `"medium"` | `{"effort": "medium", "summary": config.reasoning_summary}` |
| `"high"` | `{"effort": "high", "summary": config.reasoning_summary}` |
| `None` | No change (existing behavior) |

Existing `kwargs["reasoning"]` takes absolute precedence if set.

##### Anthropic
More nuanced — `reasoning_effort` maps to thinking mode + budget:

| `reasoning_effort` | Anthropic thinking config |
|---|---|
| `"low"` | `type="enabled", budget_tokens=4096` (minimal thinking) |
| `"medium"` | `type="adaptive"` if supported, else `type="enabled"` with model default budget |
| `"high"` | `type="adaptive"` if supported, else `type="enabled"` with generous budget |
| `None` | No change (existing behavior) |

When `reasoning_effort` is set, it implies `extended_thinking=True` internally. But `kwargs["extended_thinking"]=False` can still override (explicit opt-out). All existing kwargs paths continue to work.

##### Gemini
Maps to thinking budget:

| `reasoning_effort` | Gemini `thinking_budget` |
|---|---|
| `"low"` | `4096` |
| `"medium"` | `-1` (dynamic, model decides) |
| `"high"` | `-1` (dynamic) |
| `None` | No change (existing: `-1` dynamic default for 2.5+ models) |

Existing `request.metadata["thinking_budget"]` and `kwargs["thinking_budget"]` override.

##### vLLM
Already has `reasoning` config with effort levels — nearly identical to OpenAI. Map `request.reasoning_effort` to the existing `reasoning` param, same as OpenAI.

##### Azure OpenAI
Inherited from OpenAI — no changes needed. Note: reasoning support depends on the Azure deployment's model — older deployments may not support it.

##### Ollama
Already has `thinking_effort` config (high/medium/low) with model capability detection for deepseek-r1, qwen3, etc. Phase 2 adds `request.reasoning_effort` as an additional input with the same semantics:

| `reasoning_effort` | Ollama `think` param |
|---|---|
| `"low"` | `True` (Ollama's thinking is binary on/off — effort maps to enable) |
| `"medium"` | `True` |
| `"high"` | `True` |
| `None` | Existing behavior (config `enable_thinking` + `thinking_effort`) |

Ollama's `think` parameter is boolean — there's no graduated effort control. Any non-None `reasoning_effort` enables thinking. The existing `thinking_effort` config continues to work as before.

##### Mock
Optionally accept and ignore `reasoning_effort` for API completeness.

#### Config Naming Standardization

| Provider | Current Config Name | Phase 2 Addition |
|---|---|---|
| Anthropic | `kwargs["extended_thinking"]` | Read `request.reasoning_effort` |
| OpenAI | `reasoning` (config) / `kwargs["extended_thinking"]` | Read `request.reasoning_effort` |
| Gemini | `request.metadata["thinking_budget"]` | Read `request.reasoning_effort` |
| vLLM | `reasoning` (config) | Read `request.reasoning_effort` |
| Ollama | `thinking_effort` / `enable_thinking` (config) | Read `request.reasoning_effort` |

All existing config names continue to work. `request.reasoning_effort` is the new portable interface layered alongside them.

---

### 5. Orchestrator Touchpoints (Minimal in Phase 2)

**Decision:** Minimal, surgical orchestrator changes in Phase 2 that are required for provider improvements to be useful. Defer larger improvements to Phase 3.

**Do in Phase 2 (loop-basic only):**

**5a. Set `reasoning_effort` on ChatRequest:**
```python
chat_request = ChatRequest(
    messages=messages_objects,
    tools=tools_list,
    reasoning_effort=self.config.get("reasoning_effort"),
)
```
Also keep passing `extended_thinking` via kwargs for backward compat.

**5b. Enrich `PROVIDER_ERROR` event with LLMError fields:**
```python
except LLMError as e:
    await hooks.emit(PROVIDER_ERROR, {
        "provider": provider_name,
        "error": {"type": type(e).__name__, "msg": str(e)},
        "retryable": e.retryable,
        "status_code": e.status_code,
    })
    raise
except Exception as e:
    await hooks.emit(PROVIDER_ERROR, {
        "provider": provider_name,
        "error": {"type": type(e).__name__, "msg": str(e)},
    })
    raise
```

**Defer to Phase 3:**
- Consistent `provider:request` / `provider:response` emission across all 3 orchestrators
- `tool_choice`, `stop`, `timeout` passthrough on ChatRequest
- loop-streaming and loop-events fixes (missing events, wrong event names, sequential tools)
- Migration from `extended_thinking` kwargs to ChatRequest-only

---

### 6. Bug Fixes (While We're There)

**6a. Gemini: Missing `_repaired_tool_ids` tracking** — Gemini's `_find_missing_tool_results` can detect the same missing IDs infinitely because it lacks the `_repaired_tool_ids` set that Anthropic and OpenAI both have. Add it to prevent infinite repair loops.

**6b. Gemini: `thoughts_token_count` not mapped** — Fixed as part of Usage fields (section 3).

---

## Implementation Order

```
Tier 1 — Reference implementations:

  Step 1: OpenAI (+ Azure OpenAI inherits)
    |-- Error translation (OpenAI SDK errors -> kernel types)
    |-- Retry pattern (disable SDK retries, add Phase 2 pattern)
    |-- Usage fields (reasoning_tokens from output_tokens_details)
    |-- reasoning_effort support
    +-- Verify Azure OpenAI inherits correctly

  Step 2: Gemini
    |-- Error translation (google.api_core.exceptions -> kernel types)
    |-- Retry pattern (currently zero protection — highest priority)
    |-- Usage fields (thoughts_token_count, cached_content_token_count)
    |-- reasoning_effort support
    +-- Bug fix: _repaired_tool_ids tracking

  Step 3: Anthropic
    |-- Error translation (refactor existing catches to use kernel types)
    |-- Retry refactor (expand from RateLimitError-only to all retryable)
    |-- Usage fields (cache tokens to named fields, keep extras)
    +-- reasoning_effort support

Tier 2 — Follows from Tier 1:

  Step 4: vLLM
    |-- Error translation (reuse OpenAI SDK mapping)
    |-- Add retry pattern (connection/timeout for local server)
    |-- Usage fields (reasoning_tokens, same as OpenAI path)
    +-- reasoning_effort (already has config, add request field)

Tier 3 — Adapt existing patterns:

  Step 5: Ollama
    |-- Error translation (ResponseError -> kernel types by status_code)
    |-- Keep existing _retry_with_backoff for connection errors
    |-- Usage fields (reasoning_tokens=None, document limitation)
    +-- reasoning_effort (add request field alongside existing thinking_effort)

Tier 4 — Optional:

  Step 6: Mock
    +-- Optionally add new Usage fields to canned responses

Orchestrator:

  Step 7: loop-basic
    |-- Set reasoning_effort on ChatRequest from config
    +-- Enrich PROVIDER_ERROR event with LLMError fields
```

---

## What Each Step Changes Per Provider

### Anthropic
| Area | Current | After Phase 2 |
|---|---|---|
| Error types raised | `RuntimeError` (after retry), native SDK errors | Kernel `RateLimitError`, `AuthenticationError`, etc. |
| Retry catches | `anthropic.RateLimitError` only | Any `LLMError` with `retryable=True` |
| Retry event | `anthropic:rate_limit_retry` | `provider:retry` |
| Usage cache fields | Extras only | Named fields + extras |
| Usage reasoning | Not reported | Stays `None` (Anthropic doesn't provide separate count) |
| `reasoning_effort` | Not read | Checked, maps to thinking config |
| retry-after > max_delay | Caps and retries anyway | Raises immediately (fail fast) |

### OpenAI
| Area | Current | After Phase 2 |
|---|---|---|
| Error types raised | Native SDK errors (implicit retry) | Kernel types |
| Retry | Invisible SDK default (2 retries) | Explicit, configurable, observable (5 retries) |
| SDK max_retries | Default (2) | `0` (we handle it) |
| Retry event | None | `provider:retry` |
| Usage reasoning_tokens | Not mapped | From `output_tokens_details.reasoning_tokens` |
| `reasoning_effort` | Via `kwargs["reasoning"]` or config | Also via `request.reasoning_effort` |

### Gemini
| Area | Current | After Phase 2 |
|---|---|---|
| Error types raised | Generic `Exception` | Kernel types via gRPC/SDK mapping |
| Retry | None | Full retry pattern |
| Usage reasoning_tokens | Not mapped | From `thoughts_token_count` |
| Usage cache_read_tokens | Not mapped | From `cached_content_token_count` |
| `reasoning_effort` | Not read | Checked, maps to `thinking_budget` |
| `_repaired_tool_ids` | Missing | Added |

### Azure OpenAI
| Area | Current | After Phase 2 |
|---|---|---|
| Everything | Inherits from OpenAI | Inherits all Phase 2 changes |
| Azure-specific | N/A | Verify content filter error codes map correctly |

### vLLM
| Area | Current | After Phase 2 |
|---|---|---|
| Error types raised | Generic `Exception` | Kernel types (OpenAI SDK mapping) |
| Retry | None | Phase 2 pattern (connection/timeout focus) |
| Usage reasoning_tokens | Not mapped | From `output_tokens_details.reasoning_tokens` |
| `reasoning_effort` | Has `reasoning` config | Also via `request.reasoning_effort` |
| Token accounting | Harmony tokenizer for GPT-OSS | Unaffected (feeds into same conversion) |

### Ollama
| Area | Current | After Phase 2 |
|---|---|---|
| Error types raised | Raw `ResponseError`, `ConnectionError`, etc. | Kernel types by status_code |
| Retry | Own `_retry_with_backoff` (connection only) | Kept + kernel error translation layer |
| Usage reasoning_tokens | Not mapped | `None` (can't separate from output — documented) |
| `reasoning_effort` | Has `thinking_effort` config | Also via `request.reasoning_effort` |
| thinking detection | Model capability detection | Unchanged |

### Mock
| Area | Current | After Phase 2 |
|---|---|---|
| Usage fields | Hardcoded 3 fields | Optionally add new optional fields |

---

## Backward Compatibility

| Change | Breaking? | Why Not |
|---|---|---|
| Error types from native to kernel | Near-zero risk | Orchestrators catch `Exception`. `__cause__` preserves native. |
| OpenAI SDK retries disabled | Behavior change | From invisible 2 to visible 5. Net improvement in resilience + observability. |
| Usage gains optional fields | No | `None` defaults. Existing code unaffected. |
| `reasoning_effort` added | No | `None` = "no opinion." Existing kwargs/config override. |
| Retry event `anthropic:rate_limit_retry` renamed | Yes for specific listeners | Renamed to `provider:retry`. Document in changelog. |
| Anthropic cache tokens set as named fields | No | Extras still set too. Both paths work. |
| Ollama keeps existing retry | No | Connection retry unchanged. Error translation is additive. |
| vLLM gains retry | Behavior change | From zero retries to 5. Net improvement in resilience. |

---

## Success Criteria

### Tier 1 (must pass before merge)
- [ ] Anthropic raises kernel error types (`__cause__` preserves native errors)
- [ ] Anthropic retries all retryable errors (not just RateLimitError)
- [ ] Anthropic reports `cache_read_tokens` and `cache_write_tokens` in Usage
- [ ] Anthropic reads `request.reasoning_effort` and maps to thinking config
- [ ] OpenAI raises kernel error types
- [ ] OpenAI has explicit retry (SDK retries disabled)
- [ ] OpenAI reports `reasoning_tokens` in Usage
- [ ] OpenAI reads `request.reasoning_effort`
- [ ] Gemini raises kernel error types
- [ ] Gemini has retry pattern (no longer zero protection)
- [ ] Gemini reports `reasoning_tokens` and `cache_read_tokens` in Usage
- [ ] Gemini reads `request.reasoning_effort`
- [ ] Gemini has `_repaired_tool_ids` tracking
- [ ] All 3 emit `provider:retry` events during retries
- [ ] retry-after > max_delay raises immediately (all 3)

### Tier 2 (must pass before merge)
- [ ] Azure OpenAI inherits all OpenAI changes correctly
- [ ] vLLM raises kernel error types and has retry
- [ ] vLLM reports `reasoning_tokens` in Usage
- [ ] vLLM reads `request.reasoning_effort`

### Tier 3 (must pass before merge)
- [ ] Ollama translates `ResponseError` to kernel types
- [ ] Ollama's existing connection retry still works
- [ ] Ollama reads `request.reasoning_effort`
- [ ] Ollama documents `reasoning_tokens=None` limitation

### Orchestrator (must pass before merge)
- [ ] loop-basic sets `reasoning_effort` on ChatRequest from config
- [ ] loop-basic enriches `PROVIDER_ERROR` event with `retryable` and `status_code`

### Cross-cutting (must pass before merge)
- [ ] Existing tests pass across all repos (zero regressions)
- [ ] New tests cover error translation, retry, usage fields, and reasoning_effort per provider

---

## Open Questions (To Resolve During Implementation)

1. **Gemini's Google GenAI SDK** — does it expose explicit error classes, or only `google.api_core.exceptions`? Need to verify during implementation.

2. **Should we update loop-streaming and loop-events in Phase 2 or defer entirely to Phase 3?** Current recommendation: defer. But if the missing `PROVIDER_RESPONSE` emission makes testing impossible, we may pull it forward.

3. **Should `reasoning_effort` on ChatRequest eventually replace `extended_thinking` kwargs entirely?** Long term yes, but not in Phase 2. Phase 2 adds the new path; deprecation comes later.

4. **OpenAI `output_tokens_details.reasoning_tokens`** — is it always present for reasoning models? Need to verify against actual API responses. If absent for some models, handle gracefully (`None`).

5. **vLLM and OpenAI code duplication** — `_convert_messages()`, `_convert_to_chat_response()`, continuation loop are near-copies. Phase 2 is an opportunity to note this for future extraction, but should not block the current work. Evaluate after implementation whether a shared `openai_responses_helpers` module would be appropriate.

6. **Ollama's `ResponseError` structure** — verify that `.status_code` is reliable for classification. If Ollama doesn't consistently set status codes, fall back to message-based classification.

7. **Azure content filter error codes** — verify the specific Azure error response shape for content filter violations and ensure they map to `ContentFilterError`.
