# unified-llm-client Design

## Goal

Build a standalone, pip-installable Python library that faithfully implements StrongDM's Attractor NLSpec "Unified LLM Client" specification — a provider-agnostic LLM client with generate(), stream(), generate_object(), middleware chains, typed error hierarchy, retry with backoff, model catalog, and provider adapters for OpenAI, Anthropic, and Gemini.

## Background

StrongDM's Attractor NLSpec defines a "Unified LLM Client" specification. The Amplifier ecosystem currently achieves ~70% of this spec's requirements through its distributed modular architecture (providers + orchestrators + hooks), but the remaining 30% — especially streaming, unified error hierarchy, retry system, and model catalog — are missing. The spec's DoD scorecard for the Unified LLM layer is the lowest at 71.7%.

The directive is strict, 100% DoD per the NLSpec. The WHOLE point is to bring StrongDM's design in AS-IS, try it out, and learn from it. NOT LiteLLM. A faithful implementation of the actual spec design.

## Approach: Standalone Library, Not an Amplifier Module

### Why NOT an Amplifier module

The spec's UnifiedLLMClient does not fit any of Amplifier's 5 module types:

- **Not a Provider** — The Provider protocol has exactly 5 methods (name, get_info, list_models, complete, parse_tool_calls). The spec's interface (generate, stream, generate_object, middleware, retry) is a higher-level abstraction that CONSUMES providers.
- **Not a Tool** — It's not LLM-decided functionality.
- **Not an Orchestrator** — It doesn't drive the session loop.
- **Not a Hook** — It's not a lifecycle observer.
- **Not a Context manager** — It doesn't manage conversation state.

Creating a new module type would violate Amplifier's "small, stable, boring" kernel principle.

### Why a standalone library

- **100% spec fidelity** — The library IS the spec. Every type, function, behavior, exactly as designed.
- **No dual-path problem** — Each adapter handles both complete() and stream() with one set of request/response translation code. Bridging to Amplifier providers for complete() while going direct-to-SDK for stream() would duplicate all request translation code.
- **Self-contained & testable** — The DoD checklist maps directly to test cases. Can be tested independently of Amplifier.
- **"Try it out and learn" value** — We discover which patterns are genuinely valuable before promoting anything upstream.
- **pip-installable** — Can be used outside Amplifier too.

### How it integrates with Amplifier

The Attractor bundle's `loop-pipeline` orchestrator imports and uses it:

```python
from unified_llm import Client, generate, stream

client = Client.from_env()

result = await generate(
    model="claude-sonnet-4-20250514",
    prompt="Implement the feature",
    tools=[...],
    max_tool_rounds=3,
    max_retries=2,
    client=client,
)
```

Standard Amplifier providers remain mounted for other tools/modules. The unified-llm-client is a parallel path, not a replacement.

## Architecture

```
┌─ Attractor Bundle ─────────────────────────────────────────┐
│                                                             │
│  loop-pipeline (Amplifier Orchestrator Module)              │
│     │                                                       │
│     │ imports                                               │
│     ▼                                                       │
│  unified-llm-client (Python Library - pip installable)      │
│     │                                                       │
│     ├── Client                                              │
│     │   ├── from_env()          → auto-detect from env vars │
│     │   └── Client(adapters={}) → explicit construction     │
│     │                                                       │
│     ├── High-Level API                                      │
│     │   ├── generate()       → GenerateResult               │
│     │   ├── stream()         → StreamResult (async iter)    │
│     │   ├── generate_object()→ GenerateResult with .output  │
│     │   └── stream_object()  → partial object stream        │
│     │                                                       │
│     ├── Provider Adapters (own SDK wrappers)                │
│     │   ├── OpenAIAdapter      (openai SDK, Responses API)  │
│     │   ├── AnthropicAdapter   (anthropic SDK, Messages API)│
│     │   ├── GeminiAdapter      (google-genai SDK)           │
│     │   └── OpenAICompatAdapter(Chat Completions for vLLM…) │
│     │                                                       │
│     ├── Middleware Chain (onion pattern)                     │
│     ├── Error Hierarchy (13 typed errors)                   │
│     ├── Retry System (backoff + jitter + Retry-After)       │
│     └── Model Catalog (JSON data file)                      │
│                                                             │
│  Standard Amplifier providers still mounted for other tools │
└─────────────────────────────────────────────────────────────┘
```

## Components

### Layer 1: Provider Adapters

Each adapter wraps a provider SDK and implements the ProviderAdapter interface:

- `name` property
- `complete(request: Request) -> Response` — blocking, no retry
- `stream(request: Request) -> AsyncIterator[StreamEvent]` — streaming, no retry
- `close()` — release resources
- Optional: `initialize()`, `supports_tool_choice(mode)`

Each adapter handles:

1. System message extraction (per provider convention)
2. Message/ContentPart translation to provider format
3. Tool definition translation
4. ToolChoice mapping
5. Generation parameter mapping
6. ResponseFormat translation
7. provider_options escape hatch
8. Response → unified Response translation
9. Error → typed error hierarchy translation
10. Stream chunk → StreamEvent normalization

### Layer 2: Provider Utilities

- Retry system with exponential backoff + jitter + Retry-After header parsing
- Error hierarchy with 13 typed errors and retryability classification
- HTTP status → error type mapping
- gRPC code → error type mapping (Gemini)

### Layer 3: Core Client

- Client class with provider routing (by `provider` field or default)
- `complete(request)` and `stream(request)` — low-level, no retry
- Middleware chain execution (onion pattern, applies to both complete and stream)
- `from_env()` constructor (reads OPENAI_API_KEY, ANTHROPIC_API_KEY, GEMINI_API_KEY)
- Module-level default client via `set_default_client()`

### Layer 4: High-Level API

- `generate()` — complete with automatic tool execution loop, retry, timeout
- `stream()` — streaming with tool execution loop, retry on initial connection only
- `generate_object()` — structured output with JSON schema validation
- `stream_object()` — streaming structured output with incremental JSON parsing
- Model catalog with `get_model_info()`, `list_models()`, `get_latest_model()`

## Public API Surface

### Client Construction

```python
Client.from_env() -> Client
Client(
    providers: dict[str, ProviderAdapter],
    default_provider: str | None,
    middleware: list[Middleware] | None,
)
set_default_client(client: Client) -> None
```

### High-Level Functions

```python
generate(
    model, prompt?, messages?, system?, tools?, tool_choice?,
    max_tool_rounds=1, stop_when?, response_format?,
    temperature?, top_p?, max_tokens?, stop_sequences?,
    reasoning_effort?, provider?, provider_options?,
    max_retries=2, timeout?, abort_signal?, client?,
) -> GenerateResult

stream(same params as generate) -> StreamResult

generate_object(
    model, prompt?, schema, ...same params,
) -> GenerateResult  # result.output has parsed object

stream_object(
    model, prompt?, schema, ...same params,
) -> async iterator of partial objects
```

### Low-Level Client Methods

```python
client.complete(request: Request) -> Response    # No retry
client.stream(request: Request) -> AsyncIterator[StreamEvent]  # No retry
client.close() -> None
```

### Model Catalog

```python
get_model_info(model_id: str) -> ModelInfo | None
list_models(provider: str | None) -> list[ModelInfo]
get_latest_model(provider: str, capability: str | None) -> ModelInfo | None
```

## Data Model (30+ types)

### Core Types

| Type | Description |
|---|---|
| **Message** | role, content (List[ContentPart]), name, tool_call_id + convenience constructors (system, user, assistant, tool_result) + .text accessor |
| **Role** | SYSTEM, USER, ASSISTANT, TOOL, DEVELOPER |
| **ContentPart** | Tagged union with kind discriminator: TEXT, IMAGE, AUDIO, DOCUMENT, TOOL_CALL, TOOL_RESULT, THINKING, REDACTED_THINKING |
| **ImageData** | url \| data, media_type, detail (auto/low/high). Local file paths auto-read and base64-encode. |
| **AudioData** | url \| data, media_type |
| **DocumentData** | url \| data, media_type, file_name |
| **ToolCallData** | id, name, arguments, type |
| **ToolResultData** | tool_call_id, content, is_error, image_data, image_media_type |
| **ThinkingData** | text, signature (for round-tripping), redacted flag |

### Request/Response

| Type | Description |
|---|---|
| **Request** | model, messages, provider?, tools?, tool_choice?, response_format?, temperature?, top_p?, max_tokens?, stop_sequences?, reasoning_effort?, metadata?, provider_options? |
| **Response** | id, model, provider, message, finish_reason, usage, raw?, warnings?, rate_limit? + convenience accessors (.text, .tool_calls, .reasoning) |
| **FinishReason** | reason (stop/length/tool_calls/content_filter/error/other), raw |
| **Usage** | input_tokens, output_tokens, total_tokens, reasoning_tokens?, cache_read_tokens?, cache_write_tokens?, raw? + addition operator |
| **ResponseFormat** | type (text/json/json_schema), json_schema?, strict? |
| **Warning** | message, code? |
| **RateLimitInfo** | requests_remaining?, requests_limit?, tokens_remaining?, tokens_limit?, reset_at? |

### Generation Results

| Type | Description |
|---|---|
| **GenerateResult** | text, reasoning?, tool_calls, tool_results, finish_reason, usage, total_usage, steps, response, output? |
| **StepResult** | text, reasoning?, tool_calls, tool_results, finish_reason, usage, response, warnings |
| **StreamResult** | async iterator over StreamEvent + response(), text_stream, partial_response |
| **StreamAccumulator** | process(event), response() |

### Streaming

| Type | Description |
|---|---|
| **StreamEvent** | type, delta?, text_id?, reasoning_delta?, tool_call?, finish_reason?, usage?, response?, error?, raw? |
| **StreamEventType** | 13 types: STREAM_START, TEXT_START, TEXT_DELTA, TEXT_END, REASONING_START, REASONING_DELTA, REASONING_END, TOOL_CALL_START, TOOL_CALL_DELTA, TOOL_CALL_END, FINISH, ERROR, PROVIDER_EVENT |

### Tools

| Type | Description |
|---|---|
| **Tool** | name, description, parameters (JSON Schema), execute? (handler function) |
| **ToolChoice** | mode (auto/none/required/named), tool_name? |
| **ToolCall** | id, name, arguments, raw_arguments? |
| **ToolResult** | tool_call_id, content, is_error |

### Configuration

| Type | Description |
|---|---|
| **TimeoutConfig** | total?, per_step? |
| **AdapterTimeout** | connect (10s), request (120s), stream_read (30s) |
| **RetryPolicy** | max_retries (2), base_delay (1.0), max_delay (60.0), backoff_multiplier (2.0), jitter (true), on_retry callback? |
| **ModelInfo** | id, provider, display_name, context_window, max_output?, supports_tools, supports_vision, supports_reasoning, input_cost_per_million?, output_cost_per_million?, aliases |

## Data Flow

### generate() flow

1. Caller invokes `generate(model, prompt, tools, ...)`.
2. High-level API resolves `client` (explicit or module-level default).
3. Model string → provider routing via catalog or `provider` param.
4. Build `Request` from params.
5. **Tool loop** begins (up to `max_tool_rounds` iterations):
   a. Apply middleware chain (request phase, registration order).
   b. Adapter translates Request → provider SDK format.
   c. Adapter calls provider SDK `complete()`.
   d. Adapter translates SDK response → unified `Response`.
   e. Apply middleware chain (response phase, reverse order).
   f. If `finish_reason == tool_calls` and tools have `execute` handlers:
      - Execute tool calls, collect ToolResults.
      - Append assistant message + tool results to messages.
      - Loop back to (a).
   g. If `finish_reason == stop` or no executable tools: break.
6. **Retry** wraps step 5a-5e. On retryable error: exponential backoff + jitter, up to `max_retries`.
7. Return `GenerateResult` with all steps, accumulated usage, final text.

### stream() flow

1. Same setup as generate() steps 1-4.
2. Adapter calls provider SDK streaming endpoint.
3. Adapter yields normalized `StreamEvent` objects through middleware chain.
4. `StreamAccumulator` assembles events into a final `Response`.
5. Tool loop: if stream ends with `tool_calls`, execute tools and re-stream.
6. Only the initial connection is retried, not partial data delivery.
7. Return `StreamResult` (async iterator over events + `response()` accessor).

## Error Handling

### Error Hierarchy

```
SDKError
 ├── ProviderError (provider, status_code?, error_code?, retryable, retry_after?, raw?)
 │    ├── AuthenticationError      (401, non-retryable)
 │    ├── AccessDeniedError        (403, non-retryable)
 │    ├── NotFoundError            (404, non-retryable)
 │    ├── InvalidRequestError      (400/422, non-retryable)
 │    ├── RateLimitError           (429, retryable)
 │    ├── ServerError              (500-504, retryable)
 │    ├── ContentFilterError       (varies, non-retryable)
 │    ├── ContextLengthError       (413, non-retryable)
 │    └── QuotaExceededError       (varies, non-retryable)
 ├── RequestTimeoutError           (retryable)
 ├── AbortError                    (non-retryable)
 ├── NetworkError                  (retryable)
 ├── StreamError                   (retryable)
 ├── InvalidToolCallError          (non-retryable)
 ├── NoObjectGeneratedError        (non-retryable)
 └── ConfigurationError            (non-retryable)
```

Names deliberately avoid shadowing Python built-ins.

### Error Translation

Each adapter maps provider-specific errors to the typed hierarchy:

- **HTTP status codes** → error type (401→Authentication, 429→RateLimit, 500→Server, etc.)
- **gRPC codes** → error type (Gemini uses gRPC: UNAUTHENTICATED→Authentication, RESOURCE_EXHAUSTED→RateLimit, etc.)
- **SDK exceptions** → wrapped with original exception preserved in `raw`

### Retryability

Every error carries a `retryable` boolean. The retry system only retries errors marked retryable. Callers can also inspect this for custom retry logic.

## Retry System

- **Exponential backoff**: `delay = min(base_delay * (multiplier ^ attempt), max_delay)`
- **Jitter**: `delay *= random(0.5, 1.5)`
- **Retry-After header**: if < max_delay, use provider's delay; if > max_delay, raise immediately
- **Per-step retry**: retries apply to individual LLM calls, not entire multi-step operations
- **Streaming**: only initial connection retried; no retry after partial data delivered
- **generate_object**: LLM call retried, schema validation failures NOT retried
- **max_retries=0**: disables retries
- **Provider adapters do NOT retry** — retry lives in Layer 2, applied by Layer 4

## Middleware Chain

Onion/chain-of-responsibility pattern:

```python
def middleware(request, next):
    # Inspect/modify request
    response = next(request)
    # Inspect/modify response
    return response
```

- Registration order for request phase (first registered = first to execute)
- Reverse order for response phase
- Streaming middleware yields events through the chain
- Common uses: logging, caching, cost tracking, rate limiting, prompt injection detection, circuit breaker

## Module Structure

```
unified-llm-client/
├── pyproject.toml
├── unified_llm/
│   ├── __init__.py            # Public API exports
│   ├── client.py              # Client class, from_env(), provider routing
│   ├── types.py               # All 30+ data model types
│   ├── errors.py              # Full 13-type error hierarchy
│   ├── retry.py               # RetryPolicy, exponential backoff + jitter
│   ├── middleware.py           # Middleware chain (onion pattern)
│   ├── catalog.py             # Model catalog, get_model_info(), list_models()
│   ├── generate.py            # generate(), stream(), generate_object(), stream_object()
│   ├── adapters/
│   │   ├── __init__.py        # ProviderAdapter interface
│   │   ├── openai.py          # OpenAI Responses API adapter
│   │   ├── anthropic.py       # Anthropic Messages API adapter
│   │   ├── gemini.py          # Gemini API adapter
│   │   └── openai_compat.py   # OpenAI-compatible endpoints (vLLM, Ollama, etc.)
│   └── data/
│       └── models.json        # Shipped model catalog
├── tests/
│   ├── unit/                  # Fast, no API keys needed
│   │   ├── test_types.py
│   │   ├── test_errors.py
│   │   ├── test_retry.py
│   │   ├── test_middleware.py
│   │   └── test_catalog.py
│   ├── adapter/               # Per-adapter (mocked SDK responses)
│   │   ├── test_openai.py
│   │   ├── test_anthropic.py
│   │   └── test_gemini.py
│   └── dod/                   # One file per DoD section
│       ├── test_8_1_core_infra.py
│       ├── test_8_2_provider_adapters.py
│       ├── test_8_3_content_model.py
│       ├── test_8_4_generation.py
│       ├── test_8_5_reasoning.py
│       ├── test_8_6_caching.py
│       ├── test_8_7_tool_calling.py
│       ├── test_8_8_error_handling.py
│       ├── test_8_9_cross_provider_parity.py
│       └── test_8_10_integration_smoke.py
└── README.md
```

## Testing Strategy

### Unit Tests (no API keys)

- All 30+ types: construction, serialization, edge cases
- Error hierarchy: classification, retryability, HTTP status mapping
- Retry: delay calculation, jitter, Retry-After handling, max_retries=0
- Middleware: chain execution order, streaming middleware, error propagation
- Model catalog: lookup, filtering, unknown model passthrough

### Adapter Tests (mocked SDK responses)

- Per-adapter: request translation, response translation, error translation
- Streaming: event normalization for each provider's native format
- Edge cases: provider quirks (Anthropic beta headers, OpenAI Responses API, Gemini gRPC errors)

### DoD Tests (Sections 8.1-8.10)

Each DoD checklist item becomes a test. The 8.9 Cross-Provider Parity matrix is a parameterized test:

- 15 test cases × 3 providers = 45 matrix cells
- Includes: simple generation, streaming, image input, tool calls, parallel tools, multi-step loops, streaming with tools, structured output, reasoning tokens, error handling, usage accuracy, prompt caching, provider_options passthrough

### Integration Smoke Tests (real API keys, Section 8.10)

6 end-to-end tests against all 3 providers with real API keys:

1. Basic generation (text non-empty, usage > 0, finish_reason == "stop")
2. Streaming (concatenated deltas == response.text)
3. Tool calling with parallel execution (steps >= 2)
4. Image input (text non-empty response)
5. Structured output (parsed, validated object)
6. Error handling (NotFoundError for nonexistent model)

## Amplifier Integration Points

### loop-pipeline orchestrator update

Small update to import and use the library instead of going through Amplifier's Provider protocol for LLM calls. The orchestrator still implements the Amplifier Orchestrator protocol and participates in the session lifecycle normally.

### Hook bridge (optional, for observability)

The orchestrator can emit custom Amplifier hook events that mirror the library's operations:

- `unified-llm:generate` — before/after generate() calls
- `unified-llm:stream_start` / `unified-llm:stream_end` — streaming lifecycle
- `unified-llm:retry` — retry attempts with error details
- `unified-llm:model_routed` — provider routing decisions

This gives Amplifier's hook system visibility into the library's operations without the library depending on Amplifier.

## "Prove Then Promote" Roadmap

After the library is working and the full DoD passes:

| Pattern Proven in Library | Promotes To | Benefit |
|---|---|---|
| Typed error hierarchy | `amplifier-core` error vocabulary | All providers get consistent errors |
| Retry with backoff + jitter | Shared utility for all providers | Gemini gets retry (currently has ZERO) |
| Streaming event taxonomy | Provider protocol extension (future) | Streaming becomes a first-class capability |
| Cache/reasoning token tracking | `Usage` model expansion | All providers report cache metrics |
| Model catalog | Foundation-level utility | Centralized model metadata |
| Middleware pattern | Hook system enhancement | More flexible pre/post processing |

Each promotion is an informed decision based on what we learned, not speculation.

## Constraints and Decisions

1. **NOT LiteLLM** — Faithful to StrongDM's actual spec design.
2. **NOT an Amplifier module** — Doesn't fit any of the 5 module types; forcing it would violate both the spec and Amplifier's design.
3. **Own adapters, not bridging** — Each adapter wraps the SDK directly for both complete() and stream(), avoiding dual-path translation duplication.
4. **Spec-first, not Amplifier-first** — The library implements the spec AS-IS; Amplifier integration is a thin layer on top.
5. **Standard Amplifier providers remain mounted** — For other tools and modules that need them.
6. **OpenAI uses Responses API** — Not Chat Completions (per spec requirement).
7. **Advisory model catalog** — Unknown model strings pass through (not restrictive).
8. **Undefined spec types** — StopCondition (callable), AbortSignal (follows web platform pattern), Timestamp (datetime) will be defined during implementation.

## Open Questions

1. **Repo location**: New repo `amplifier-module-unified-llm-client`? Or subdirectory of `amplifier-bundle-attractor`? (Leaning toward own repo for pip-installability.)
2. **SDK version pinning**: Which versions of openai, anthropic, google-genai SDKs to target?
3. **Async-only or sync wrappers too**: Spec mentions sync wrappers for convenience. Include or defer?
