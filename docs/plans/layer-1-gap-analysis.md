# Layer 1 Gap Analysis: Unified LLM Client vs Amplifier Provider System

**Date:** 2026-02-07
**Status:** Analysis Complete
**Scope:** Attractor `unified-llm-spec.md` (128 requirements) vs Amplifier provider system (amplifier-core + 7 provider modules + 3 orchestrators + 11 hook modules)
**Method:** Source code audit of all 17 cloned repositories, parallel agent investigation with cross-validation

---

## Executive Summary

The Attractor unified-llm-spec defines a comprehensive multi-provider LLM client library with 128 concrete requirements across 14 categories. After deep inspection of Amplifier's actual source code, we find:

- **~70% is already covered**, often in a more powerful form than the Attractor specifies
- **~20% can be achieved by improving existing modules** with no new architecture
- **~10% requires design work** (primarily the streaming protocol, deferred to Layer 2)
- **All changes are backward-compatible** with zero breakage to existing callers
- **Total kernel changes: ~65 lines** across 2 existing files plus 1 new file

The core insight is that Amplifier's modular architecture (kernel + providers + orchestrators + hooks) distributes the Attractor's monolithic client capabilities across multiple composable layers. Most "gaps" are not missing capabilities but capabilities that exist in a different shape or need to be promoted from ad-hoc patterns to first-class features.

---

## Architectural Context

### The Shape Difference

The Attractor spec defines a **monolithic client library**:

```
Application -> UnifiedLLMClient.generate(model, messages, tools) -> Response
                    |
                    +-- Tool loop, retries, streaming, middleware all inside
```

Amplifier distributes this across **composable modules**:

```
Application -> Orchestrator.execute(prompt)
                    |
                    +-- Provider.complete(ChatRequest) -> ChatResponse
                    +-- Tool execution (parallel, with hooks)
                    +-- Hooks: pre/post interception, approval, injection
                    +-- Context: message history management
```

This means:
- The Attractor's `generate()` with tool loop = Amplifier's Orchestrator
- The Attractor's `complete()` / `stream()` = Amplifier's Provider
- The Attractor's middleware chain = Amplifier's Hook system
- The Attractor's error handling + retries = distributed across Provider + Orchestrator + Hooks

**Neither shape is wrong.** They're different architectural choices. For Layer 3 (the pipeline engine), we may want a thin `generate()` convenience wrapper that combines Amplifier's primitives into a single call, but the underlying capabilities are present.

---

## What We Already Have (Code-Verified)

### Provider Protocol and Message Model

The kernel defines a clean provider contract in `amplifier_core/interfaces.py`:

```python
class Provider(Protocol):
    @property
    def name(self) -> str: ...
    def get_info(self) -> ProviderInfo: ...
    async def list_models(self) -> list[ModelInfo]: ...
    async def complete(self, request: ChatRequest, **kwargs) -> ChatResponse: ...
    def parse_tool_calls(self, response: ChatResponse) -> list[ToolCall]: ...
```

Key data structures (all with `model_config = ConfigDict(extra="allow")`):

| Type | Fields | Notes |
|------|--------|-------|
| `ChatRequest` | messages, tools, response_format, temperature, top_p, max_output_tokens, conversation_id, stream, metadata | 9 declared fields, extensible via extra="allow" |
| `ChatResponse` | content, tool_calls, usage, degradation, finish_reason, metadata | 6 declared fields |
| `Message` | role (6 values), content (str or list[ContentBlockUnion]), name, tool_call_id, metadata | Union content type |
| `Usage` | input_tokens, output_tokens, total_tokens | 3 required fields, extensible |
| `ContentBlockUnion` | TextBlock, ThinkingBlock, RedactedThinkingBlock, ToolCallBlock, ToolResultBlock, ImageBlock, ReasoningBlock | 7 block types, discriminated union |

### Native API Usage (Verified)

Each provider uses the vendor's native API, not compatibility shims:

| Provider | API | SDK |
|----------|-----|-----|
| Anthropic | Messages API (`/v1/messages`) | `AsyncAnthropic` |
| OpenAI | Responses API (`/v1/responses`) | `AsyncOpenAI` with `client.responses.create()` |
| Gemini | Native GenAI (`generateContent`) | `google.genai.Client` |

This matches the Attractor's PS-01, PS-02, PS-03 requirements exactly.

### Thinking and Reasoning (Verified)

All three providers handle thinking/reasoning natively:

| Provider | Trigger | Content Block | Preservation |
|----------|---------|---------------|--------------|
| Anthropic | `kwargs["extended_thinking"]` | `ThinkingBlock(thinking, signature)` | Signature round-tripped verbatim |
| OpenAI | `kwargs["extended_thinking"]` or `request.reasoning` | `ThinkingBlock(content=[encrypted_content, reasoning_id])` | Encrypted state preserved across turns |
| Gemini | `request.metadata["thinking_budget"]` | `ThinkingBlock(thinking, signature)` | Thought signature preserved |

Amplifier's thinking block model is **ahead of the Attractor spec** in that it already handles:
- Anthropic's `signature` field for multi-turn verification
- OpenAI's `encrypted_content` for stateless reasoning preservation
- Gemini's dynamic budget (`-1` = model decides)
- `RedactedThinkingBlock` for vendor-redacted thinking data

### Prompt Caching (Verified — Anthropic)

The Anthropic provider implements comprehensive prompt caching with three injection points:

1. **System message**: Last content block gets `cache_control: {type: ephemeral}`
2. **Tools**: Last tool definition gets `cache_control`
3. **Messages**: Last content block of last message gets `cache_control`

Cache tokens are reported in Usage via `extra="allow"`:
- `cache_creation_input_tokens` (from Anthropic response)
- `cache_read_input_tokens` (from Anthropic response)

OpenAI and Gemini have automatic caching but don't surface metrics in Usage yet.

### Structured Output (Verified)

Full support via `ResponseFormat` union type:

```python
ResponseFormatText       # type="text"
ResponseFormatJson       # type="json" (any JSON)
ResponseFormatJsonSchema # type="json_schema", json_schema=dict, strict=bool|None
```

Plus `Degradation` reporting when a provider can't honor the requested format — the Attractor spec has no equivalent.

### Hook System as Middleware (Verified)

The hook system provides 5 actions with defined precedence:

| Priority | Action | Effect |
|----------|--------|--------|
| 1 (highest) | `deny` | Block operation, short-circuit chain |
| 2 | `ask_user` | Request user approval (transforms to deny/continue) |
| 3 | `inject_context` | Add content to agent's conversation (multiple merge) |
| 4 | `modify` | Transform event data (chains through handlers) |
| 5 (lowest) | `continue` | No-op, proceed normally |

**Evidence these are exercised in production:**

- **`inject_context`**: Used by `hooks-progress-monitor` (ships with foundation) to inject paralysis warnings. Uses `ephemeral=True` + `append_to_last_tool_result=True` + `context_injection_role="user"`.
- **`ask_user`**: Fully implemented in `coordinator.process_hook_result()` with approval flow, timeout, fail-closed default. Routes to `ApprovalProvider` protocol.
- **`deny`**: Used by `hooks-approval`, `hooks-scheduler-cost-aware`, `hooks-scheduler-heuristic`.
- **`modify`**: Used by scheduler hooks to redirect tool selections.

Provider-related events emitted by orchestrators:

| Event | loop-basic | loop-streaming | loop-events |
|-------|-----------|----------------|-------------|
| `provider:request` | Every iteration | Every iteration | Only at max_iterations |
| `provider:response` | Yes | No | No |
| `tool:pre` | Yes | Yes | Yes |
| `tool:post` | Yes | Yes | Yes |

Foundation example `18_custom_hooks.py` demonstrates provider-level middleware patterns:
- `PerformanceMonitor` on `provider:post` tracking token usage from response
- `CostTracker` on `provider:post` doing real-time cost estimation with per-model pricing
- `RateLimiter` on `tool:pre` with sliding window
- Multiple hooks on the same event composing cleanly

### Retry (Verified — Anthropic Only)

The Anthropic provider has a mature custom retry system (120 lines):
- SDK retries explicitly disabled (`max_retries=0`)
- Custom loop: up to 5 retries with exponential backoff (1s, 2s, 4s, 8s, 16s)
- Retry-after header honoring (uses server's delay when available)
- Jitter: +/-20% randomness
- Max delay cap: 60s (configurable)
- Observability: `anthropic:rate_limit_retry` and `anthropic:rate_limited` hook events
- Well-factored into 3 extractable methods: `_parse_rate_limit_info()`, `_calculate_retry_delay()`, and the retry loop itself

OpenAI uses implicit SDK retries (default 2, invisible to Amplifier). Gemini has zero retry protection.

### Per-Request Model Override (Verified)

All three providers support per-request model override via `kwargs["model"]`:
- Anthropic: `kwargs.get("model", self.default_model)`
- OpenAI: `kwargs.get("model", self.default_model)`
- Gemini: `kwargs.get("model", self.default_model)`

The capability exists at the provider layer but orchestrators don't currently pass it through — they only pass `extended_thinking=True`.

### Tool Execution (Verified)

| Aspect | loop-basic | loop-streaming | loop-events |
|--------|-----------|----------------|-------------|
| Parallel execution | Yes (`asyncio.gather`) | Yes (`asyncio.gather`) | Sequential (`for` loop) |
| Tool repair safety net | Via provider | Via provider | Via provider |
| hook gating (deny) | `tool:pre` | `tool:pre` | `tool:pre` + `tool:selecting` |
| Error handling | Caught internally, returned as error strings | Same | Same (with `continue`) |

All three providers implement tool repair safety nets that scan for orphaned tool calls and inject synthetic error results.

---

## Improvements Required

### Phase 1: Kernel Vocabulary (~65 lines, enables everything)

These are additive changes to existing Pydantic models. All use `None` defaults. Zero breakage.

#### 1a. Promote Usage Extra Fields to Named Fields

**File:** `amplifier_core/message_models.py`
**Effort:** ~5 lines

```python
class Usage(BaseModel):
    model_config = ConfigDict(extra="allow")
    input_tokens: int
    output_tokens: int
    total_tokens: int
    # Promote from hidden extra fields to named fields:
    reasoning_tokens: int | None = None
    cache_read_tokens: int | None = None
    cache_write_tokens: int | None = None
```

**Rationale:** Anthropic already passes `cache_creation_input_tokens` and `cache_read_input_tokens` as extras. Making them named fields makes them discoverable by hooks and orchestrators. Meets the two-implementation rule (2+ providers have reasoning tokens, 2+ have caching).

**Backward compatibility:** All existing `Usage(input_tokens=X, output_tokens=Y, total_tokens=Z)` calls continue to work unchanged.

#### 1b. Add Standard ChatRequest Fields

**File:** `amplifier_core/message_models.py`
**Effort:** ~10 lines

```python
class ChatRequest(BaseModel):
    model_config = ConfigDict(extra="allow")
    # ... existing fields ...
    # New optional fields:
    model: str | None = None
    tool_choice: str | dict[str, str] | None = None
    stop: list[str] | None = None
    reasoning_effort: str | None = None
    timeout: float | None = None
```

**Rationale:** Providers already read these from `**kwargs`. Making them first-class fields means:
- Hooks on `provider:request` can see and modify them via `modify` action
- Orchestrators can set them based on configuration
- The contract is explicit rather than implicit

**Backward compatibility:** All existing `ChatRequest(messages=..., tools=...)` calls continue to work. Providers that don't read these fields ignore them (extra="allow" already passes unknown fields through).

#### 1c. Add Error Vocabulary

**File:** New file `amplifier_core/llm_errors.py` (or section in existing `errors.py`)
**Effort:** ~50 lines

```python
class LLMError(Exception):
    """Base for all LLM provider errors."""
    def __init__(self, message: str, *, provider: str | None = None,
                 status_code: int | None = None, retryable: bool = False):
        super().__init__(message)
        self.provider = provider
        self.status_code = status_code
        self.retryable = retryable

class RateLimitError(LLMError):
    """Provider rate limit exceeded."""
    def __init__(self, message: str, *, retry_after: float | None = None, **kwargs):
        super().__init__(message, retryable=True, **kwargs)
        self.retry_after = retry_after

class AuthenticationError(LLMError):
    """Invalid or missing API credentials."""
    pass

class ContextLengthError(LLMError):
    """Request exceeds model's context window."""
    pass

class ContentFilterError(LLMError):
    """Content blocked by provider's safety filter."""
    pass

class ProviderUnavailableError(LLMError):
    """Provider service unavailable (5xx, network error)."""
    def __init__(self, message: str, **kwargs):
        super().__init__(message, retryable=True, **kwargs)

class LLMTimeoutError(LLMError):
    """Request timed out."""
    def __init__(self, message: str, **kwargs):
        super().__init__(message, retryable=True, **kwargs)
```

**Rationale:** Providers currently raise native SDK errors that can't be caught cross-provider. This vocabulary enables:
- Retry hooks that catch `RateLimitError` regardless of which provider raised it
- Fallback orchestrators that catch `ProviderUnavailableError` and try another provider
- Cost hooks that catch `ContextLengthError` and suggest a model with a larger window

**Backward compatibility:** New exception classes that nothing references yet. Existing `except Exception` catches continue to work. Providers adopt incrementally — a provider that still raises native errors continues to work.

### Phase 2: Provider Improvements (Improve Existing Modules)

#### 2a. Extract Shared Retry Utility

**Source:** Anthropic provider's `_calculate_retry_delay()` and retry loop
**Destination:** Shared utility (in core or a lightweight shared package)

Extract the generic pieces:
- `calculate_retry_delay(retry_after, attempt, min_delay, max_delay, jitter)` — already fully generic in Anthropic
- `async_retry_on_rate_limit(callable, max_retries, ...)` — structural wrapper

Each provider supplies a provider-specific callback:
- `extract_retry_after(error) -> float | None` — parses headers from native error

**Apply to providers:**
- **Gemini:** Currently has zero retry protection. Add shared retry wrapper. Highest risk to fix.
- **OpenAI:** Currently uses implicit SDK retries (2, invisible). Disable SDK retries (`max_retries=0`), add shared wrapper for observability parity.
- **Anthropic:** Refactor to use shared utility. Same behavior, cleaner code.

**Observability:** Standardize event names: `provider:rate_limit_retry` (replacing `anthropic:rate_limit_retry`) so hooks can observe retries from any provider.

#### 2b. Standardize Error Translation

Each provider wraps native SDK errors in kernel vocabulary:

| Native Error | Kernel Type |
|---|---|
| `anthropic.RateLimitError` | `amplifier_core.RateLimitError` |
| `anthropic.AuthenticationError` | `amplifier_core.AuthenticationError` |
| `openai.RateLimitError` | `amplifier_core.RateLimitError` |
| `openai.AuthenticationError` | `amplifier_core.AuthenticationError` |
| `google.api_core.exceptions.ResourceExhausted` | `amplifier_core.RateLimitError` |
| Any provider's 413 / context length | `amplifier_core.ContextLengthError` |
| Any provider's content filter | `amplifier_core.ContentFilterError` |
| `asyncio.TimeoutError` | `amplifier_core.LLMTimeoutError` |

Translation uses `raise KernelType(...) from native_error` to preserve the original exception chain.

#### 2c. Surface Cache and Reasoning Tokens in Usage

| Provider | Current State | Improvement |
|---|---|---|
| Anthropic | Already passes `cache_creation_input_tokens`, `cache_read_input_tokens` as extras | Map to new named fields `cache_write_tokens`, `cache_read_tokens` |
| OpenAI | Reports `input_tokens`, `output_tokens` only | Add `reasoning_tokens` from Responses API `output_tokens_details.reasoning_tokens` |
| Gemini | Reports basic token counts only | Add `reasoning_tokens` from `thoughtsTokenCount` if available |

#### 2d. Consistent Thinking Trigger Convention

Currently inconsistent:
- Anthropic: `kwargs["extended_thinking"]`
- OpenAI: `kwargs["extended_thinking"]` or `request.reasoning` (reads extra field)
- Gemini: `request.metadata["thinking_budget"]`

After Phase 1b adds `reasoning_effort` to ChatRequest, providers read from the standardized field:
- All providers check `request.reasoning_effort` first
- Fall back to `kwargs["extended_thinking"]` for backward compatibility
- Gemini additionally checks `request.metadata["thinking_budget"]` for backward compatibility

### Phase 3: Orchestrator Consistency (Improve Existing Modules)

#### 3a. Consistent Hook Event Emission

All three orchestrators should emit `provider:request` before every LLM call and `provider:response` after every LLM call. This ensures provider-level hooks (cost tracking, cache observation, retry middleware) work regardless of which orchestrator is active.

**Changes needed:**
- `loop-streaming`: Add `provider:response` emission after `provider.complete()` returns
- `loop-events`: Add `provider:request` emission on every iteration (not just max_iterations). Add `provider:response` emission.

#### 3b. Pass ChatRequest Fields Through

Currently all three orchestrators build ChatRequest with only `messages` and `tools`:

```python
chat_request = ChatRequest(messages=messages_objects, tools=tools_list)
```

After Phase 1b, they should also pass through fields from configuration or hook modifications:

```python
chat_request = ChatRequest(
    messages=messages_objects,
    tools=tools_list,
    model=self.config.get("model"),           # from orchestrator config
    tool_choice=self.config.get("tool_choice"),
    reasoning_effort=self.config.get("reasoning_effort"),
    # ... etc
)
```

This enables hooks on `provider:request` to see and modify these fields via the `modify` action.

#### 3c. Support Hook-Driven Model Routing

With `provider:request` emitted consistently and `model` on ChatRequest, a hook can:

```python
async def route_model(event: str, data: dict) -> HookResult:
    # Read complexity signal from the request
    if is_simple_request(data):
        data["model"] = "claude-haiku-3"
    else:
        data["model"] = "claude-sonnet-4-5"
    return HookResult(action="modify", data=data)
```

The orchestrator passes the (potentially modified) `model` field through to `provider.complete()` via kwargs. No orchestrator code change beyond 3b is needed — the hook system handles the routing.

### Phase 4: Graduate Patterns to Modules

#### 4a. Cost Tracker Module

Graduate the `CostTracker` pattern from `amplifier-foundation/examples/18_custom_hooks.py` into a proper `hooks-cost-tracker` module:

- Registers on `provider:response`
- Reads `Usage` (including new named fields for reasoning/cache tokens)
- Applies per-model pricing from configurable rate table
- Accumulates session-level costs
- Can `deny` on `provider:request` when budget exceeded
- Emits `cost:update` events for observability

#### 4b. Improve Scheduler Hooks for Provider-Level Routing

The existing `hooks-scheduler-cost-aware` and `hooks-scheduler-heuristic` operate only on `tool:selecting`. Extend to also register on `provider:request` for model-level routing:

- Cost-aware scheduler: Route to cheaper models when budget is tightening
- Heuristic scheduler: Route based on request complexity signals

### Phase 5: Streaming Protocol (Deferred to Layer 2 Analysis)

The streaming protocol is the one area requiring genuine design work. The nlspec defines:

- 12 streaming event types with start/delta/end pattern
- Normalized SSE parsing across providers
- `stream()` method on the client returning `AsyncIterator<StreamEvent>`
- `StreamAccumulator` bridging streaming and blocking modes

Amplifier currently has:
- `ChatRequest.stream = True` (field exists, no provider reads it)
- `content_block:start/delta/end` events defined in core
- `loop-streaming` checks for `provider.stream()` but no provider implements it

**Why defer:** The Coding Agent Loop spec (Layer 2) and Pipeline Engine spec (Layer 3) will define exactly what streaming contract the pipeline nodes need. Designing the streaming protocol now without that context risks building the wrong interface.

**Likely direction:** Add an optional `stream()` method to the Provider protocol. Providers that implement it return an `AsyncIterator`. Providers that don't fall back to `complete()`. The orchestrator adapts based on availability (loop-streaming already has this pattern).

---

## Coverage Matrix

### By Attractor Requirement Category

| Category | Total Reqs | Already Covered | After Phase 1-4 | Remaining | Notes |
|----------|-----------|-----------------|------------------|-----------|-------|
| Provider Support | 10 | 8 | 10 | 0 | Native APIs verified |
| Core API | 10 | 7 | 8 | 2 | stream(), stream_object() deferred |
| Message/Content Model | 17 | 15 | 17 | 0 | 7 block types, extras pass through |
| Streaming | 10 | 3 | 3 | 7 | Deferred to Layer 2 |
| Prompt Caching | 8 | 5 | 8 | 0 | Anthropic complete, others after Phase 2c |
| Reasoning Tokens | 8 | 6 | 8 | 0 | After Usage fields + consistent triggers |
| Structured Output | 6 | 6 | 6 | 0 | Fully covered + degradation reporting |
| Error Handling | 15 | 3 | 14 | 1 | After taxonomy + provider improvements |
| Middleware/Interceptors | 4 | 4 | 4 | 0 | Hook system exceeds spec requirements |
| Model Catalog | 8 | 5 | 7 | 1 | Per-provider exists. Cross-provider is app-layer |
| Configuration | 10 | 8 | 10 | 0 | Different shape (mount plan), equivalent capability |
| Tool/Function Calling | 16 | 13 | 15 | 1 | tool_choice after Phase 1b. repair_tool_call is a nice-to-have |
| Cost/Usage Tracking | 9 | 5 | 9 | 0 | After Usage fields + cost tracker module |
| Concurrency | 9 | 6 | 8 | 1 | Per-request timeout after Phase 1b. AbortSignal via CancellationToken |
| **TOTAL** | **128** | **~90 (70%)** | **~115 (90%)** | **~13 (10%)** | Remaining is primarily streaming |

### Amplifier Advantages Not in Attractor Spec

These are capabilities Amplifier has that the Attractor's unified-llm-spec does not define:

1. **5-action hook system** with precedence ordering (deny > ask_user > inject_context > modify > continue) — more powerful than the spec's onion middleware
2. **Degradation reporting** — explicit `ChatResponse.degradation` when format falls back
3. **Multi-provider simultaneous mount** — orchestrator gets all providers, not single-client routing
4. **Thinking block preservation rules** — signatures, encrypted state, vendor-specific round-tripping
5. **Approval gates** — human-in-the-loop as a composable module
6. **`extra="allow"` philosophy** — all Pydantic models accept arbitrary fields, making vendor-specific features work without kernel changes
7. **Auto-continuation** — OpenAI provider handles incomplete responses transparently (up to 5 attempts)
8. **Tool repair safety nets** — all 3 providers scan for orphaned tool calls
9. **Ephemeral context injection** — hooks can inject one-shot messages that don't persist in history
10. **Session-level cost tracking** — `SessionStatus.estimated_cost` at the kernel level

---

## Backward Compatibility Analysis

### Principle

Every change follows the same pattern:
- **New optional fields** with `None` defaults on existing models
- **New exception types** that nothing catches yet
- **New events** that nothing listens for yet
- **Internal refactoring** with identical external behavior

No existing caller, module, or configuration breaks.

### Change-by-Change Verification

| Change | Type | Breaking? | Why Not |
|--------|------|-----------|---------|
| Add 3 fields to `Usage` | Optional fields, `None` default | No | Existing `Usage(input_tokens=X, ...)` unchanged |
| Add 5 fields to `ChatRequest` | Optional fields, `None` default | No | Existing `ChatRequest(messages=..., tools=...)` unchanged |
| Add error taxonomy | New exception classes | No | Nothing catches them yet. `except Exception` still works |
| Extract retry utility | Internal refactor | No | Same behavior, different code structure |
| Add retry to Gemini/OpenAI | Improved resilience | No | Callers see fewer errors, never more |
| Translate errors to kernel types | Provider internal | Near-zero risk | `raise X from native_error` preserves chain |
| Emit more hook events | Additive events | No | No hook is harmed by events it doesn't listen for |
| Pass more ChatRequest fields | More data on request | No | Providers ignore fields they don't read (extra="allow") |

### The One Subtle Case

If a provider translates `anthropic.RateLimitError` to `amplifier_core.RateLimitError`, code that catches `anthropic.RateLimitError` directly would miss it. Mitigation:
- No Amplifier module catches native SDK errors — they catch generic `Exception`
- `raise X from native_error` preserves the original via `__cause__`
- Providers can optionally raise as a subclass if needed

---

## Implementation Phases and Effort

### Phase 1: Kernel Vocabulary
**Effort:** ~2 hours
**Risk:** Near zero (additive optional fields + new file)
**Repos touched:** `amplifier-core` only

- 1a: Add 3 optional fields to Usage (~5 lines)
- 1b: Add 5 optional fields to ChatRequest (~10 lines)
- 1c: Add error vocabulary (~50 lines in new file)
- Tests for new fields and error types

### Phase 2: Provider Improvements
**Effort:** ~1-2 days
**Risk:** Low (internal refactoring with same external behavior)
**Repos touched:** `amplifier-module-provider-anthropic`, `amplifier-module-provider-openai`, `amplifier-module-provider-gemini`

- 2a: Extract shared retry utility, apply to all 3 providers
- 2b: Add error translation wrappers in all 3 providers
- 2c: Surface cache/reasoning tokens in Usage for OpenAI and Gemini
- 2d: Standardize thinking trigger to use `request.reasoning_effort`

### Phase 3: Orchestrator Consistency
**Effort:** ~1 day
**Risk:** Low (additive event emission + field pass-through)
**Repos touched:** `amplifier-module-loop-basic`, `amplifier-module-loop-streaming`, `amplifier-module-loop-events`

- 3a: Consistent `provider:request` and `provider:response` emission
- 3b: Pass ChatRequest fields through from config
- 3c: Hook-driven model routing (no code change beyond 3b — hook system handles it)

### Phase 4: Module Graduation
**Effort:** ~1 day
**Risk:** Low (new modules, nothing depends on them yet)
**Repos touched:** New modules or improvements to existing scheduler modules

- 4a: Graduate cost tracker from example to module
- 4b: Extend scheduler hooks for provider-level routing

### Phase 5: Streaming Protocol
**Effort:** TBD (design work, informed by Layer 2 analysis)
**Risk:** Medium (new protocol surface)
**Repos touched:** `amplifier-core` (protocol), all providers (implementation), orchestrators (consumption)

- Deferred until Layer 2 (Coding Agent Loop) analysis reveals what the pipeline engine actually needs

---

## Dependencies and Ordering

```
Phase 1 (kernel vocabulary)
    |
    +---> Phase 2 (provider improvements)
    |         |
    |         +---> Phase 2a (retry) depends on Phase 1c (error types)
    |         +---> Phase 2c (usage) depends on Phase 1a (usage fields)
    |         +---> Phase 2d (thinking) depends on Phase 1b (chatrequest fields)
    |
    +---> Phase 3 (orchestrator consistency)
    |         |
    |         +---> Phase 3b depends on Phase 1b (chatrequest fields)
    |
    +---> Phase 4 (module graduation)
              |
              +---> Phase 4a (cost tracker) depends on Phase 1a + Phase 3a
              +---> Phase 4b (scheduler) depends on Phase 3a + Phase 3b

Phase 5 (streaming) is independent — informed by Layer 2 analysis
```

**Critical path:** Phase 1 -> Phase 2a + 2b (retry + errors) -> Phase 3a (consistent events)

This unlocks the most capability with the least effort and enables the cost/routing modules.

---

## What This Means for the Attractor Pipeline (Layer 3)

After Phases 1-4, the Amplifier provider system will support everything the Attractor's Coding Agent Loop (Layer 2) and Pipeline Engine (Layer 3) need from the LLM layer, with the exception of streaming.

For the pipeline engine specifically:

| Pipeline Need | How Amplifier Provides It |
|---|---|
| Call an LLM with a prompt | `provider.complete(ChatRequest)` or `orchestrator.execute(prompt)` |
| Get structured output | `ChatRequest.response_format = ResponseFormatJsonSchema(...)` |
| Track token costs | Usage with named fields + cost tracker hook |
| Retry on failure | Shared retry utility in each provider + error taxonomy for hooks |
| Route to different models per node | `ChatRequest.model` + hook on `provider:request` with `modify` |
| Control reasoning effort per node | `ChatRequest.reasoning_effort` |
| Human-in-the-loop gates | Hook with `ask_user` action, handled by coordinator |
| Inject context between stages | Hook with `inject_context` action, ephemeral or persistent |
| Observe all LLM interactions | Hooks on `provider:request` / `provider:response` |

The one remaining question is whether pipeline nodes need **streaming output** from providers. If yes, Phase 5 becomes a prerequisite for Layer 3. If nodes can work with complete responses (which the Attractor's own `CodergenBackend.run()` suggests — it returns `String | Outcome`), then streaming can be added later as an optimization.

---

## Next Steps

1. **Review this analysis** for alignment with Amplifier philosophy and priorities
2. **Proceed to Layer 2 analysis** (Coding Agent Loop vs Amplifier Orchestrator + Tools) — this will also inform Phase 5 (streaming) design decisions
3. **Begin Phase 1 implementation** when ready — it's the enabling work for everything else
