---
type: API Reference
title: Streaming and Async Patterns
description: Core streaming APIs across Runnables and chat models, including stream/astream entry points, event types, async iteration patterns, message chunk composition, and protocol-level event handling.
tags: [streaming, async, iterator, astream, stream, events, chat-models, runnables]
verified:
  - by: openwiki/0.5.0
    at: 2026-11-15T08:15:26.682Z
sources:
  - id: openwiki-source-c9313cf42f0120d86b20245f
    resource: repo://libs/core/langchain_core/callbacks/base.py
  - id: openwiki-source-5f8bc32563177d89fbab9b2f
    resource: repo://libs/core/langchain_core/language_models/chat_model_stream.py
  - id: openwiki-source-c52037e7b642f7ac5a7642a8
    resource: repo://libs/core/langchain_core/language_models/chat_models.py
  - id: openwiki-source-77dc1fb726463969f9d53658
    resource: repo://libs/core/langchain_core/messages/ai.py
  - id: openwiki-source-a1981e868973f6fd7f71e12e
    resource: repo://libs/core/langchain_core/runnables/base.py
generated: { by: "openwiki/0.5.0", at: "2026-11-15T08:15:26.682Z" }
---

## Overview

This reference documents streaming and asynchronous patterns in LangChain Core. Streaming enables incremental, real-time output delivery from any `Runnable`, with specialized support in chat models for token-by-token delivery and tool call finalization.

The streaming system operates at two levels:

1. **Runnable Protocol** (`Runnable.stream()` / `Runnable.astream()`) – Generic streaming for any Runnable, yielding complete outputs one at a time as they are produced
2. **Chat Model Streaming** (`BaseChatModel.stream()` / `BaseChatModel.astream()`) – Specialized token/chunk streaming with callback events and chunk composition
3. **Protocol Events** (`Runnable.astream_events()` with `version="v3"`) – Structured event streaming with typed projections and content-block-level granularity

---

## Runnable Streaming

### Runnable.stream()

**Location**: `repo://libs/core/langchain_core/runnables/base.py#L1258-L1277`

Synchronous streaming entry point that yields outputs as they are produced.

```python
def stream(
    self,
    input: Input,
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> Iterator[Output]:
    """Yield outputs as they are produced."""
```

**Behavior**:

- **Default implementation**: Calls `invoke()` once and yields a single result. This ensures all Runnables provide a streaming interface even if not optimized for it.
- **Subclass optimization**: Subclasses override to yield partial outputs as they are produced (e.g., chat models streaming tokens, chains streaming intermediate results).
- **Configuration threading**: Accepts optional `RunnableConfig` for tags, metadata, callbacks, and other execution options.
- **Callback integration**: Fires callback events for each yielded chunk; streaming callbacks (if attached) trigger this method automatically.

**Usage**:

```python
# Generic Runnable
for output in runnable.stream(input):
    print(output)

# Chat model (yields AIMessageChunk objects)
for chunk in model.stream(messages):
    print(chunk.content, end="", flush=True)
```

---

### Runnable.astream()

**Location**: `repo://libs/core/langchain_core/runnables/base.py#L1279-L1298`

Asynchronous variant of `stream()`, mirroring the synchronous logic using async/await.

```python
async def astream(
    self,
    input: Input,
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> AsyncIterator[Output]:
    """Yield outputs asynchronously as they are produced."""
```

**Behavior**:

- **Default implementation**: Calls `ainvoke()` once and yields a single result asynchronously.
- **Subclass optimization**: Subclasses override for native async streaming (e.g., async generators, async HTTP iterators).
- **Fallback to sync**: If a subclass implements only `stream()`, the default `astream()` can fall back via `run_in_executor()`.

**Usage**:

```python
# Generic Runnable
async for output in runnable.astream(input):
    print(output)

# Chat model (yields AIMessageChunk objects)
async for chunk in model.astream(messages):
    print(chunk.content, end="", flush=True)
```

---

## Chat Model Streaming

Chat models implement specialized streaming with fine-grained token delivery and tool call support.

### BaseChatModel.stream()

**Location**: `repo://libs/core/langchain_core/language_models/chat_models.py#L770-L898`

Synchronous streaming entry point that yields `AIMessageChunk` objects as they are produced by the model.

```python
def stream(
    self,
    input: LanguageModelInput,
    config: RunnableConfig | None = None,
    *,
    stop: list[str] | None = None,
    **kwargs: Any,
) -> Iterator[AIMessageChunk]:
```

**Behavior**:

- **Streaming Decision**: Checks `_should_stream(async_api=False)` to determine if streaming is implemented and enabled
- **Fallback**: If streaming is not available, delegates to `invoke()` and yields a single result cast to `AIMessageChunk`
- **Callback Setup**: Configures callbacks via `CallbackManager.configure()`
- **Event Lifecycle**:
  - Fires `on_chat_model_start` before beginning iteration
  - For each chunk from `_stream()`:
    - Sets a unique run ID (prefixed with `LC_ID_PREFIX`) if not already present
    - Computes response metadata via `_gen_info_and_msg_metadata()`
    - Transforms content to content blocks if `output_version="v1"`
    - Fires `on_llm_new_token` with the token and full chunk object
    - Yields the chunk immediately
    - Accumulates the chunk in memory for later merging
  - Yields a final empty chunk with `chunk_position="last"` if not already yielded by the model; this signals stream completion and triggers tool call finalization
  - On success, fires `on_llm_end` with a merged result containing all accumulated chunks
  - On exception, fires `on_llm_error` with partial accumulation before re-raising

**Rate Limiting**: Acquires a permit from `self.rate_limiter` (if attached) before beginning; blocks until available.

**Parameters**:

- `input`: Messages, message dict, or other model input
- `config`: Execution configuration (tags, metadata, callbacks, run ID, etc.)
- `stop`: Optional stop sequences
- `**kwargs`: Additional model-specific arguments

**Returns**: Iterator of `AIMessageChunk` objects

---

### BaseChatModel.astream()

**Location**: `repo://libs/core/langchain_core/language_models/chat_models.py#L901-L1033`

Asynchronous variant of `stream()`, mirroring the synchronous logic but using async/await and `AsyncCallbackManager`.

```python
async def astream(
    self,
    input: LanguageModelInput,
    config: RunnableConfig | None = None,
    *,
    stop: list[str] | None = None,
    **kwargs: Any,
) -> AsyncIterator[AIMessageChunk]:
```

**Key Differences**:

- Checks `_should_stream(async_api=True)`; can fall back to sync `_stream()` if only that is implemented
- Uses `await` for callback events (`await run_manager.on_llm_new_token(...)`, `await run_manager.on_llm_end(...)`)
- Iterates via `async for chunk in self._astream(...)`
- Acquires rate limit via `await self.rate_limiter.aacquire(blocking=True)`

The async streaming protocol is identical to sync: initialize callbacks, yield chunks immediately, fire `on_llm_new_token` per token, finalize tool call chunks on the "last" signal.

---

## Streaming Control Flow

### _should_stream()

**Location**: `repo://libs/core/langchain_core/language_models/chat_models.py#L590-L626`

Determines whether a given model call should use streaming. Returns `True` if:

1. **Streaming is implemented**: The model overrides `_stream()` (for sync) or `_astream()` (for async)
2. **Streaming is not explicitly disabled** via:
   - `self.disable_streaming is True`
   - `self.disable_streaming == "tool_calling"` and `tools` are passed
   - `stream=False` in call kwargs
   - `self.streaming is False` on the instance
3. **At least one affirmative trigger**:
   - Explicit `stream=True` kwarg in the call
   - Instance-level `streaming=True` attribute
   - A v1 streaming callback handler (`_StreamingCallbackHandler`) is attached to the run manager

**Signature**:

```python
def _should_stream(
    self,
    *,
    async_api: bool,
    run_manager: CallbackManagerForLLMRun | AsyncCallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> bool:
```

---

### _streaming_disabled()

**Location**: `repo://libs/core/langchain_core/language_models/chat_models.py#L566-L588`

Returns `True` if streaming is hard-disabled for a call, regardless of implementation availability or handler attachment. Respected by both `_should_stream()` and `_should_use_protocol_streaming()`.

Checks:

- `self.disable_streaming is True`
- `self.disable_streaming == "tool_calling"` with `tools` passed
- `stream=<falsy>` in call kwargs
- `self.streaming is False` on the instance

---

## Message Chunks and Composition

### AIMessageChunk

**Location**: `repo://libs/core/langchain_core/messages/ai.py#L418-L536`

Represents a partial, incremental update to an AI message during streaming. Unlike `AIMessage`, it is designed to be **mergeable** via the `+` operator.

**Fields**:

- **`content`**: String or list of content blocks (delta for that step). During streaming, contains only the new token(s) or content delta; accumulates across chunks via concatenation.
- **`tool_call_chunks`**: List of `ToolCallChunk` objects representing incomplete tool calls being streamed. Arguments are accumulated and parsed incrementally via `parse_partial_json()`.
- **`chunk_position`**: Optional sentinel; when set to `"last"`, indicates the final chunk in the stream. Triggers finalization of `tool_call_chunks` into complete `tool_calls` and `invalid_tool_calls` when aggregated.
- **`response_metadata`**: Model-specific metadata (latency, model_provider, usage counters, finish reason, etc.). Merged across chunks, with usage counts summed.

**Merging via the + Operator**:

The `+` operator (implemented via `add_ai_message_chunks()`) combines two chunks:

1. **Content**: Text content concatenated; structured content blocks merged by block type
2. **Tool Call Arguments**: Arguments from `tool_call_chunks` appended progressively, enabling incremental JSON parsing
3. **Metadata**: Response metadata and usage counts merged; later values override except for usage, which is summed
4. **Chunk Position**: If any operand has `chunk_position="last"`, result marks position as "last", triggering tool call finalization
5. **Message ID**: Selected by rank: provider-assigned (non-`LC_*` prefixed) > `LC_run_*` > `lc_*` auto IDs

**Tool Call Finalization**:

When `chunk_position="last"` is present or set during a merge, the validator `init_tool_calls()` automatically:

- Finalizes all accumulated tool call chunks via `parse_partial_json()` and schema validation
- Promotes `tool_call_chunks` to complete `tool_calls`
- Records invalid or unparseable calls in `invalid_tool_calls`

**Chunk Types**:

| Type | Use Case |
|------|----------|
| `AIMessageChunk` | Emitted by chat models during streaming |
| `HumanMessageChunk` | Incremental user input (e.g., streaming transcription) |
| `SystemMessageChunk` | Incremental system prompt updates |
| `ToolMessageChunk` | Incremental tool results from streaming APIs |

All inherit from `BaseMessageChunk` and support merging via `+`.

---

## Protocol-Level Streaming: ChatModelStream

Protocol-level streaming (`astream_events(version="v3")`) enables structured, content-block-centric event handling with typed projections for text, reasoning, tool calls, and usage.

### ChatModelStream

**Location**: `repo://libs/core/langchain_core/language_models/chat_model_stream.py#L1190-L1300`

Synchronous wrapper for protocol-level event handling and typed projections. Returned by `BaseChatModel.stream_events(version="v3")`.

**Initialization**:

```python
def __init__(
    self,
    *,
    namespace: list[str] | None = None,
    node: str | None = None,
    message_id: str | None = None,
) -> None:
```

**Core Properties**:

- **`namespace`**: Graph namespace path for this message
- **`node`**: Graph node that produced this message
- **`message_id`**: Stable message identifier (set by the stream driver after `on_chat_model_start`)
- **`done`**: Whether the stream has finished
- **`has_events`**: Whether any protocol events have been recorded
- **`output_message`**: The assembled `AIMessage` if the stream has finished, else `None`

**Event Dispatch**:

```python
def dispatch(self, event: Mapping[str, Any]) -> None:
```

Routes a protocol event (`message-start`, `content-block-start`, `content-block-delta`, `content-block-finish`, `message-finish`, `error`) to the appropriate internal handler. Public entry point for feeding events into the stream.

**Typed Projections** (replaying subsets of events):

Streaming builds three typed accumulators as protocol events arrive. Each projection supports both delta iteration and final value consumption:

- **`.text`**: `SyncTextProjection`
  - Iterates text deltas: `for delta in stream.text: print(delta, end="")`
  - Drains to final string: `final_text = stream.text.get()`

- **`.reasoning`**: `SyncTextProjection`
  - Similar to `.text`; accumulates reasoning blocks when present

- **`.tool_calls`**: `SyncProjection[list[ToolCall]]`
  - Iterates finalized tool call chunks: `for chunk in stream.tool_calls: ...`
  - Drains to final list: `calls = stream.tool_calls.get()`

- **`.output`**: `AIMessage`
  - Blocks until stream finishes, then returns the assembled message
  - Usage metadata available via `msg.usage_metadata`

**Usage Pattern**:

```python
stream = model.stream_events(messages, version="v3")

# Iterate text deltas
for token in stream.text:
    print(token, end="", flush=True)

# Or drain for final values
final_text = stream.text.get()
final_calls = stream.tool_calls.get()
final_message = stream.output  # Blocks until stream finishes

# Or collect all events for replay
for event in stream:
    print(event)
```

---

### AsyncChatModelStream

**Location**: `repo://libs/core/langchain_core/language_models/chat_model_stream.py#L1346-L1454`

Asynchronous variant of `ChatModelStream`. Returned by `BaseChatModel.astream_events(version="v3")`.

**Typed Projections** (async variants):

- **`.text`**: `AsyncTextProjection`
  - Iterate: `async for delta in stream.text: ...`
  - Await final: `final_text = await stream.text`
  - Or drain: `final_text = stream.text.get()` (use in sync context if manually driven)

- **`.reasoning`**: `AsyncTextProjection`
  - Same as `.text`

- **`.tool_calls`**: `AsyncProjection[list[ToolCall]]`
  - Iterate: `async for chunk in stream.tool_calls: ...`
  - Await final: `calls = await stream.tool_calls`

- **`.output`**: `AsyncProjection[AIMessage]`
  - Iterate finalized content blocks: `async for block in stream.output: ...`
  - Await final message: `msg = await stream.output`
  - Usage metadata available via `msg.usage_metadata`

**Usage Pattern**:

```python
async for chunk in model.astream_events(messages, version="v3"):
    stream = chunk  # Already a ChatModelStream or AsyncChatModelStream

# Iterate text deltas
async for token in stream.text:
    print(token, end="", flush=True)

# Or await for final values
final_text = await stream.text
final_calls = await stream.tool_calls
final_message = await stream.output
```

---

## Async Iteration and Projections

### SyncProjection

**Location**: `repo://libs/core/langchain_core/language_models/chat_model_stream.py#L219-L282`

Synchronous iterable of accumulated deltas with pull-based backpressure.

**Methods**:

- **`__iter__()`**: Create a new cursor at position 0 that pulls from the producer via `_request_more` until exhausted or an error occurs
- **`get()`**: Drain the projection and return the final accumulated value; blocks internally until completion
- **`set_start(cb)`**: Install a lazy-start callback invoked on first consumption
- **`set_request_more(cb)`**: Install the pull callback the iterator uses to drive the source; callback returns `True` when a new event was produced, `False` when exhausted

**Multiple independent iterators**: Each call to `__iter__()` creates a new cursor; all replay the full delta buffer from the start.

**Backpressure Pattern**:

The projection does not precompute or buffer all events. Instead, it uses pull-based backpressure: the iterator calls `_request_more()` on-demand to drive the source, allowing the producer to control pacing and avoid overload.

---

### SyncTextProjection

**Location**: `repo://libs/core/langchain_core/language_models/chat_model_stream.py#L284-L338`

String-specialized sync projection for `.text` and `.reasoning` projections. Enforces string-only deltas and final values.

**Additional Methods**:

- **`__str__()`**: Drain and return the full accumulated string
- **`__bool__()`**: Return whether any deltas have been pushed
- **`__repr__()`**: Return repr of accumulated text so far

---

### AsyncProjection

**Location**: `repo://libs/core/langchain_core/language_models/chat_model_stream.py#L345-L437`

Async iterable of accumulated deltas that is also awaitable for the final value. Uses `asyncio.Event` for inter-task signaling.

**Methods**:

- **`__aiter__()`**: Return an async iterator over deltas; each iterator is a new cursor at position 0
- **`__await__()`**: Await the final accumulated value; awakens on `push()` or `complete()`
- **`set_start(cb)`**: Install a lazy-start callback invoked on first consumption
- **`set_arequest_more(cb)`**: Install an async pull callback; returns `True` when new event produced, `False` when exhausted

**Dual consumption**: Consumers can iterate deltas, await the final value, or both simultaneously via independent cursors on the shared replay buffer.

**Event-Driven Pattern**:

Unlike `SyncProjection`'s pull-based backpressure, `AsyncProjection` uses event-driven signaling via `asyncio.Event`. Multiple async tasks can await the same projection, and all are awakened when new data arrives or completion is signaled.

---

### AsyncTextProjection

**Location**: `repo://libs/core/langchain_core/language_models/chat_model_stream.py#L514-L549`

String-specialized async projection. Enforces string-only deltas; `__await__` returns string or empty string if unfinished.

---

## Callback Integration

### Streaming Callbacks

**Location**: `repo://libs/core/langchain_core/callbacks/base.py#L65-L88`

Streaming fires token-level callbacks as chunks arrive, enabling real-time observation without buffering the full response.

**on_llm_new_token**

Fired once per chunk during streaming (v1 path via `stream()` / `astream()`):

```python
def on_llm_new_token(
    self,
    token: str | list[str | dict[str, Any]],
    *,
    chunk: GenerationChunk | ChatGenerationChunk | None = None,
    run_id: UUID,
    parent_run_id: UUID | None = None,
    tags: list[str] | None = None,
    **kwargs: Any,
) -> Any:
```

**Parameters**:

- **`token`**: The new token content (string) or list of content blocks (when `output_version="v1"`)
- **`chunk`**: The `ChatGenerationChunk` containing the full delta
- **`run_id`**: The ID of the current run
- **`parent_run_id`**: The parent run ID (for nested traces)
- **`tags`**: Tags for filtering and organization

**on_stream_event**

Fired once per protocol event during v3 streaming (via `stream_events(version="v3")` / `astream_events(version="v3")`):

```python
def on_stream_event(
    self,
    event: MessagesData,
    *,
    run_id: UUID,
    parent_run_id: UUID | None = None,
    tags: list[str] | None = None,
    **kwargs: Any,
) -> Any:
```

Fires once per `MessagesData` event — `message-start`, per-block `content-block-start` / `content-block-delta` / `content-block-finish`, and `message-finish`. Boundaries are explicit; a single chunk may map to multiple events.

### Streaming Callback Handlers

**`_StreamingCallbackHandler`** (v1 streaming)

**Location**: `repo://libs/core/langchain_core/tracers/_streaming.py`

Base class for handlers that observe token-level callbacks during `stream()` / `astream()`. Presence of such a handler on the run manager triggers streaming even without `stream=True` kwarg.

**`_V2StreamingCallbackHandler`** (v3 protocol streaming)

**Location**: `repo://libs/core/langchain_core/tracers/_streaming.py`

Base class for handlers that observe protocol-level events. Routes streaming through the v2 event path in `_should_use_protocol_streaming()`.

---

## Implementation Hooks for Subclasses

### _stream()

**Base Location**: `repo://libs/core/langchain_core/language_models/chat_models.py`

Subclasses override `_stream()` to produce `ChatGenerationChunk` objects. Base class provides a fallback that raises `NotImplementedError`.

**Signature**:

```python
def _stream(
    self,
    messages: list[BaseMessage],
    stop: list[str] | None = None,
    run_manager: CallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> Iterator[ChatGenerationChunk]:
```

**Yielding Chunks**:

Each chunk must contain a `ChatGenerationChunk.message` (an `AIMessageChunk`). Content is optional; the stream driver will:

1. Assign a unique run ID if not present
2. Compute and attach response metadata
3. Fire `on_llm_new_token` callbacks
4. Yield immediately to the caller

---

### _astream()

**Base Location**: `repo://libs/core/langchain_core/language_models/chat_models.py`

Asynchronous variant of `_stream()`. Subclasses override to produce `ChatGenerationChunk` objects asynchronously.

**Signature**:

```python
async def _astream(
    self,
    messages: list[BaseMessage],
    stop: list[str] | None = None,
    run_manager: AsyncCallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> AsyncIterator[ChatGenerationChunk]:
```

If only `_stream()` is implemented, async calls can fall back to running sync in a thread pool (via `run_in_executor`).

---

### _stream_chat_model_events()

**Optional Hook**: Implement for native protocol event generation (v3 streaming).

**Signature**:

```python
def _stream_chat_model_events(
    self,
    messages: list[BaseMessage],
    *,
    stop: list[str] | None = None,
    run_manager: CallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> Iterator[MessagesData]:
```

When implemented, the stream driver calls this instead of bridging `_stream()` via `chunks_to_events()`. Enables providers to emit protocol events natively without intermediate chunk conversion.

---

### _astream_chat_model_events()

**Optional Hook**: Async variant of `_stream_chat_model_events()`.

---

## Error Handling and Edge Cases

### Fallback Behavior

If a model does not implement streaming (`_should_stream()` returns `False`), `stream()` and `astream()` automatically delegate to `invoke()` or `ainvoke()` and yield a single result cast to `AIMessageChunk`. This ensures all models provide a consistent streaming interface.

### Tool Call Finalization

Tool calls are emitted as `tool_call_chunks` during streaming and finalized on the "last" signal:

1. **During streaming**: `tool_call_chunks` contain partial data (incremental JSON arguments)
2. **On "last" signal**: Chunks are parsed via `parse_partial_json()` and promoted to complete `tool_calls`
3. **On merge**: Merging two chunks with `+` respects `chunk_position="last"` to finalize any accumulated tool calls

If no explicit "last" chunk is yielded by the model, `stream()` and `astream()` yield one automatically.

### Error Propagation

On exception during streaming:

1. `on_llm_error` is fired with partial accumulation (chunks received so far)
2. The exception is re-raised to the caller
3. The stream terminates abnormally; no "last" signal is sent

---

## Configuration and Behavior Flags

### disable_streaming

**Field**: `disable_streaming: bool | Literal["tool_calling"] = False`

Controls whether streaming is disabled for calls to this model:

- `True`: Streaming is always disabled
- `"tool_calling"`: Streaming is disabled only when `tools` are passed
- `False` (default): Streaming is not disabled; other conditions determine whether it is used

---

### streaming

**Field**: `streaming: bool | None = None`

Instance-level flag to enable or disable streaming by default:

- `True`: Streaming is enabled by default (unless disabled)
- `False`: Streaming is disabled for all calls
- `None` (default): Streaming is determined per-call via `_should_stream()`

---

### output_version

**Field**: `output_version: Literal["v0", "v1"] = "v0"`

Controls whether content is stored in provider-specific format or standardized format:

- `"v0"`: Provider-specific format in content; standard format available via `content_blocks` property
- `"v1"`: Standardized format in content; consistent with `content_blocks`

When `output_version="v1"`, `stream()` and `astream()` transform content blocks and index them for consistent access.

---

## Related Concepts

- **[Runnables](/openwiki/reference/runnables.md)** – Runnable composition and streaming through chains
- **[Chat Models](/openwiki/reference/chat-models.md)** – Chat model API and specializations
- **[Callbacks](/openwiki/reference/callbacks.md)** – Callback handler architecture and lifecycle
- **[Messages](/openwiki/reference/messages.md)** – Message types and chunk composition
