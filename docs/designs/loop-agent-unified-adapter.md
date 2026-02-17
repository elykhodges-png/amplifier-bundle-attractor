# UnifiedProviderAdapter for loop-agent Design

## Goal

Integrate the `unified-llm-client` library into loop-agent's spawned agent sessions via an adapter pattern, so that all LLM calls benefit from unified error handling, retry logic, and streaming normalization — without modifying `agent_session.py`.

## Background

The Attractor pipeline's preferred execution path spawns child agent sessions (`session.spawn`) that run `loop-agent`. These child agents use Amplifier's standard `Provider.complete()` and `Provider.stream()` for LLM calls — bypassing the `unified-llm-client` library entirely. This means:

- No unified error hierarchy (retry with backoff, typed errors)
- No streaming event normalization across providers
- No spec-compliant behavior guarantees
- The pipeline's Path B (direct tool loop) uses unified-llm-client, but Path A (spawn → agent) does not

For the Attractor to fully run real DOT pipelines with 100% NLSpec compliance, the agent sessions spawned by the pipeline must also use the unified-llm-client for their LLM calls.

## Approach

**Adapter pattern — zero changes to `agent_session.py`.**

The `agent_session.py` file (1112 lines) contains a sophisticated state machine with multi-turn conversation history management, tool execution loops (parallel + sequential), steering and follow-up queues, loop detection, cancellation support, rich event emission (20+ event types), subagent management, and context window monitoring. This is Amplifier's value-add on top of raw LLM calls. We don't want to rewrite it or even modify it.

Instead, we create a thin adapter that satisfies the existing duck-type contract the agent expects from its provider:

- `adapter.complete(request: ChatRequest) -> ChatResponse`
- `adapter.stream(request: ChatRequest) -> AsyncIterator[dict]` (async generator yielding dict chunks)

Internally, the adapter translates types, calls unified-llm-client, translates results back, and maps errors.

### What this preserves

- Agent's state machine, event emission, tool loop — all untouched
- All 30+ existing test files pass unchanged (they mock the provider duck-type)
- Hook bridge middleware can be injected into the unified-llm Client inside the adapter

### What this adds

- unified-llm-client's retry with exponential backoff + jitter + Retry-After
- Typed error hierarchy with retryability classification
- Normalized streaming across all 3 providers
- Spec-compliant request/response translation
- Model catalog for model discovery

## Architecture

```
loop-agent (AgentSession — UNCHANGED)
    │
    ├── _call_provider_complete(request: ChatRequest)
    │       └── adapter.complete(request)
    │               ├── Translates ChatRequest → unified_llm.Request
    │               ├── Calls client.complete(ulm_request)
    │               ├── Translates unified_llm.Response → ChatResponse
    │               └── Maps SDKError → LLMError on failure
    │
    └── _call_provider_streaming(request: ChatRequest)
            └── adapter.stream(request)
                    ├── Translates ChatRequest → unified_llm.Request
                    ├── Calls client.stream(ulm_request)
                    ├── Translates StreamEvent → dict chunks
                    │   (with tool-call buffering for incremental → complete)
                    └── Maps SDKError → LLMError on failure
```

### Module Location

The adapter lives in the loop-agent module itself:

```
amplifier_module_loop_agent/unified_provider_adapter.py
```

This is the loop-agent's concern — how it talks to LLMs. The adapter is injected by the `AgentOrchestrator` at construction time.

### Injection Point

In `AgentOrchestrator.execute()` (`__init__.py`), after selecting the provider:

```python
provider = providers[provider_name]
# Wrap with unified-llm adapter if available
if unified_llm_available:
    from .unified_provider_adapter import UnifiedProviderAdapter
    provider = UnifiedProviderAdapter(provider_name=provider_name, model=model)
```

The adapter creates its own `unified_llm.Client` internally (via `Client.from_env()` or the provider-copy pattern with hook bridge middleware).

### Streaming Detection

The agent checks `inspect.isasyncgenfunction(provider.stream)` at construction time. The adapter's `stream()` method must be a real `async def` with `yield` (an async generator function) for this check to pass.

## Components

### Request Translation: ChatRequest → unified_llm.Request

The adapter translates Amplifier's `ChatRequest` to `unified_llm.Request`:

| ChatRequest field | unified_llm.Request field | Translation |
|---|---|---|
| `messages: list[Message]` | `messages: list[Message]` | Each Message's content blocks → ContentParts (TextBlock→TEXT, ThinkingBlock→THINKING, etc.) |
| `tools: list[ToolSpec]` | Not used — agent handles tools | Tools are NOT passed to unified-llm (the agent owns the tool loop) |
| `tool_choice: str` | Not used | Same — agent manages tool_choice |
| `reasoning_effort: str` | `reasoning_effort: str` | Direct passthrough |
| `model: str` (from config) | `model: str` | Direct passthrough |

**Key insight**: The agent owns the tool loop, so we do NOT pass tools to `unified_llm.generate()`. We only use `client.complete()` (single LLM call, no tool loop) since the agent manages the multi-turn tool execution itself.

### Response Translation (Non-Streaming): unified_llm.Response → ChatResponse

| unified_llm.Response field | ChatResponse field | Translation |
|---|---|---|
| `response.message.content` (list[ContentPart]) | `content: list[ContentBlockUnion]` | ContentPart(TEXT) → TextBlock, ContentPart(THINKING) → ThinkingBlock (preserving signature), ContentPart(TOOL_CALL) → included in tool_calls |
| `response.tool_calls` (accessor) | `tool_calls: list[ToolCall]` | unified_llm.ToolCall → amplifier_core.ToolCall |
| `response.usage` | `usage: Usage` | Map fields: input_tokens, output_tokens, total_tokens, reasoning_tokens, cache_read/write_tokens |
| `response.finish_reason` | Not directly mapped | Agent doesn't use finish_reason from ChatResponse |

### Response Translation (Streaming): StreamEvent → dict chunks

unified-llm yields typed `StreamEvent` objects. Loop-agent expects dict chunks. The mapping:

| StreamEvent.type | Dict chunk key | Notes |
|---|---|---|
| TEXT_DELTA | `{"content": event.delta}` | Direct text delta |
| REASONING_START/DELTA | `{"thinking": event.reasoning_delta}` | Reasoning content |
| REASONING_END | `{"reasoning_signature": ...}` | Signature if present (for Anthropic round-tripping) |
| TOOL_CALL_START | Buffer — don't yield yet | Start accumulating tool call |
| TOOL_CALL_DELTA | Buffer — accumulate args | Append to buffered tool call arguments |
| TOOL_CALL_END | `{"tool_calls": [buffered_call]}` | Yield complete tool call |
| FINISH | `{"usage": {...}}` | Usage data from finish event |
| STREAM_START, TEXT_START, TEXT_END | Skip | No equivalent in agent's chunk format |

**Tool-call buffering** is the key complexity. unified-llm delivers tool calls incrementally (START → DELTA → END) but the agent expects complete tool calls in one chunk. The adapter must buffer until TOOL_CALL_END.

### Error Mapping: SDKError → LLMError

| unified_llm error | amplifier_core error | retryable |
|---|---|---|
| `AuthenticationError` | `AuthenticationError` | False |
| `AccessDeniedError` | `LLMError(retryable=False)` | False |
| `NotFoundError` | `LLMError(retryable=False)` | False |
| `InvalidRequestError` | `LLMError(retryable=False)` | False |
| `RateLimitError` | `RateLimitError` | True |
| `ServerError` | `ProviderUnavailableError` | True |
| `ContentFilterError` | `ContentFilterError` | False |
| `ContextLengthError` | `ContextLengthError` | False |
| `RequestTimeoutError` | `LLMTimeoutError` | True |
| `NetworkError` | `ProviderUnavailableError` | True |
| `StreamError` | `StreamError` | True |
| `AbortError` | `LLMError(retryable=False)` | False |
| `ConfigurationError` | `LLMError(retryable=False)` | False |

The agent's `process_input()` catches `LLMError` and branches on `.retryable`:

- `retryable=False` → fatal_error → CLOSED → re-raise
- `retryable=True` → re-raise (expects external retry)

With the unified-llm adapter, retry happens INSIDE the adapter (unified-llm-client's retry system handles it before the error reaches the agent). So most retryable errors will be resolved before the agent sees them. If retries are exhausted, the error propagates as a non-retryable error.

## Data Flow

### Non-Streaming Path

```
AgentSession._call_provider_complete(ChatRequest)
  → UnifiedProviderAdapter.complete(ChatRequest)
    → translate_request(ChatRequest) → unified_llm.Request
    → client.complete(ulm_request)  [unified-llm handles retries internally]
    → translate_response(unified_llm.Response) → ChatResponse
  → AgentSession processes ChatResponse (tool calls, content, usage)
```

### Streaming Path

```
AgentSession._call_provider_streaming(ChatRequest)
  → UnifiedProviderAdapter.stream(ChatRequest)  [async generator]
    → translate_request(ChatRequest) → unified_llm.Request
    → client.stream(ulm_request)
    → for each StreamEvent:
        → translate_stream_event(event) → dict chunk (or buffer for tool calls)
        → yield chunk
  → AgentSession processes dict chunks (emits ASSISTANT_TEXT_DELTA events, etc.)
```

### Hook Bridge Integration

The adapter creates a `unified_llm.Client` with hook bridge middleware injected:

```python
class UnifiedProviderAdapter:
    def __init__(self, provider_name, model, hooks=None):
        middleware = []
        if hooks:
            from amplifier_module_loop_pipeline.hook_bridge import create_hook_bridge
            middleware.append(create_hook_bridge(hooks, lambda: {"node_id": "agent"}))

        base = Client.from_env()
        self._client = Client(
            providers=base._providers,
            default_provider=provider_name,
            middleware=middleware
        )
        self._model = model
        self._provider_name = provider_name
```

This gives the agent session full Amplifier hook observability through the same hook bridge used by the pipeline.

## Error Handling

### Retry Behavior

With the unified-llm adapter, retry with exponential backoff + jitter + Retry-After header support is handled internally by the unified-llm-client before errors propagate to the agent. This means:

- **Rate limits**: automatically retried with backoff before the agent ever sees a `RateLimitError`
- **Server errors / network errors**: automatically retried before the agent sees `ProviderUnavailableError`
- **Timeouts**: automatically retried before the agent sees `LLMTimeoutError`

If all retries are exhausted, the error propagates to the agent as the appropriate `LLMError` subclass. The agent's existing error handling then takes over (fatal_error → CLOSED → re-raise for non-retryable, re-raise for retryable).

### Edge Cases

**ReasoningBlock signature round-tripping.** Anthropic's ThinkingBlock has a `.signature` field that must survive round-trips for multi-turn conversations. The adapter must:

- On response translation: preserve `ContentPart(THINKING).thinking.signature` → `ThinkingBlock.signature`
- On request translation: preserve `ThinkingBlock.signature` → `ContentPart(THINKING).thinking.signature`

**Tool-call buffering in streaming.** If the stream terminates unexpectedly (error/cancellation) while a tool call is being buffered, the partial tool call must be discarded — not yielded as incomplete.

**Multiple tool calls in one response.** The LLM may return multiple tool calls. In streaming, each gets its own START→DELTA→END sequence. The adapter yields each complete tool call as a separate chunk, OR batches them if they arrive in the same stream segment.

## Testing Strategy

### Unit tests for the adapter (new)

- Request translation: ChatRequest → unified_llm.Request (messages, reasoning_effort, model)
- Response translation: unified_llm.Response → ChatResponse (text, thinking, tool_calls, usage)
- Streaming translation: StreamEvent sequence → dict chunks (with tool-call buffering)
- Error mapping: each SDKError subclass → correct LLMError subclass with correct retryability
- Streaming detection: `isasyncgenfunction(adapter.stream)` returns True

### Existing tests (unchanged)

All 30+ existing test files should pass unchanged — they mock `provider.complete()` and `provider.stream()` which the adapter satisfies.

### Integration tests (new)

- End-to-end: adapter + real unified-llm mock → agent session completes a multi-turn conversation
- Error propagation: adapter error → agent state machine handles correctly
- Streaming: adapter streaming → agent emits correct ASSISTANT_TEXT_DELTA events

## Configuration

The adapter is opt-in. Controlled by:

1. **Presence of unified-llm-client package** — if not installed, agent falls back to standard provider
2. **Bundle config flag** — `unified_llm_enabled: true` in orchestrator config (optional)
3. **Environment** — adapter uses `Client.from_env()` so it needs the same API key env vars

## Open Questions

1. **Should the adapter be the default or opt-in?** If unified-llm-client is installed, should the adapter always be used, or should it require explicit configuration?
2. **Hook bridge reuse** — The adapter imports `create_hook_bridge` from loop-pipeline. This creates a cross-module dependency. Should the hook bridge be moved to a shared location, or is this import acceptable?
3. **Model override** — The pipeline passes `provider_preferences` with model info to spawn. How does the adapter receive the model name? From the spawn config, from the bundle YAML, or from the agent's orchestrator config?
