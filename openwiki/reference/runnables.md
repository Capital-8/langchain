---
type: "API Reference"
title: "Runnable Protocol: Method Reference"
description: "Complete reference for Runnable protocol methods, lifecycle, and configuration. Documents synchronous and asynchronous execution, batching, streaming, composition operators, and configuration threading."
tags: [runnable, lcel, api-reference, execution, protocol]
verified:
  - by: openwiki/0.5.0
    at: 2026-11-15T08:15:26.682Z
sources:
  - id: openwiki-source-a1981e868973f6fd7f71e12e
    resource: repo://libs/core/langchain_core/runnables/base.py
generated: { by: "openwiki/0.5.0", at: "2026-11-15T08:15:26.682Z" }
---

# Runnable Protocol: Method Reference

This page provides a detailed API reference for the `Runnable` protocol in LangChain Core. The `Runnable` abstract base class defines the unified interface for synchronous and asynchronous execution, batching, streaming, and schema introspection that all LangChain components implement.

## Core Execution Methods

### Synchronous Single Invocation

**`invoke(input, config=None, **kwargs) → Output`** (`repo://libs/core/langchain_core/runnables/base.py#L949-L970`)

The abstract method that all subclasses must implement. Transforms a single input into an output synchronously.

```python
@abstractmethod
def invoke(
    self,
    input: Input,
    config: RunnableConfig | None = None,
    **kwargs: Any,
) -> Output:
    """Transform a single input into an output."""
```

**Parameters:**
- `input`: The data to transform.
- `config`: Optional `RunnableConfig` dictionary supporting:
  - `tags`: List of strings for tracing and filtering.
  - `metadata`: Dictionary of metadata for observability.
  - `callbacks`: List of callback handlers for lifecycle events.
  - `max_concurrency`: Maximum parallel operations (for batch/stream methods).
  - `run_name`: Custom name for this execution.
  - `run_id`: Explicitly set run identifier.
  - `configurable`: Dictionary of configurable parameter overrides.
  - `recursion_limit`: Maximum recursion depth (default: 25).
- `**kwargs`: Additional keyword arguments passed through to implementation.

**Returns:** Transformed output.

**Requirement:** Must be implemented by all `Runnable` subclasses.

### Asynchronous Single Invocation

**`ainvoke(input, config=None, **kwargs) → Coroutine[Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L972-L993`)

Async variant of `invoke`. Default implementation delegates to `invoke` via `run_in_executor` (thread pool).

```python
async def ainvoke(
    self,
    input: Input,
    config: RunnableConfig | None = None,
    **kwargs: Any,
) -> Output:
    """Transform a single input into an output asynchronously."""
```

**Parameters:** Same as `invoke`.

**Returns:** Coroutine yielding transformed output.

**Optimization:** Override for native async implementations (e.g., async HTTP clients, async libraries).

### Batch Processing

**`batch(inputs, config=None, *, return_exceptions=False, **kwargs) → list[Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L995-L1043`)

Processes multiple inputs in parallel. Default implementation calls `invoke` for each input using a thread pool executor.

```python
def batch(
    self,
    inputs: list[Input],
    config: RunnableConfig | list[RunnableConfig] | None = None,
    *,
    return_exceptions: bool = False,
    **kwargs: Any | None,
) -> list[Output]:
    """Process multiple inputs in parallel."""
```

**Parameters:**
- `inputs`: List of inputs to process.
- `config`: Single `RunnableConfig` (applied to all inputs) or list matching length of inputs.
- `return_exceptions`: If `True`, exceptions are returned in output list. If `False` (default), raises on first exception.
- `**kwargs`: Additional arguments passed to each `invoke` call.

**Returns:** List of outputs, preserving input order.

**Optimization:** Override for batching-aware backends (e.g., LLM batch APIs, database batch operations).

### Asynchronous Batch Processing

**`abatch(inputs, config=None, *, return_exceptions=False, **kwargs) → Coroutine[list[Output]]`** (`repo://libs/core/langchain_core/runnables/base.py#L1130-L1176`)

Async variant of `batch`. Default implementation calls `ainvoke` concurrently with `asyncio.gather`, respecting `max_concurrency` from config.

```python
async def abatch(
    self,
    inputs: list[Input],
    config: RunnableConfig | list[RunnableConfig] | None = None,
    *,
    return_exceptions: bool = False,
    **kwargs: Any | None,
) -> list[Output]:
    """Process multiple inputs concurrently."""
```

**Parameters:** Same as `batch`.

**Returns:** Coroutine yielding list of outputs.

**Concurrency Control:** `max_concurrency` from the first config limits parallelism via `asyncio.Semaphore`. No limit if not set.

### Batch as Completed (Streaming Results)

**`batch_as_completed(inputs, config=None, *, return_exceptions=False, **kwargs) → Iterator[tuple[int, Output | Exception]]`** (`repo://libs/core/langchain_core/runnables/base.py#L1065-L1129`)

Synchronous batch that yields `(index, output)` tuples as they complete, allowing early processing of results without waiting for all inputs.

```python
def batch_as_completed(
    self,
    inputs: Sequence[Input],
    config: RunnableConfig | Sequence[RunnableConfig] | None = None,
    *,
    return_exceptions: bool = False,
    **kwargs: Any | None,
) -> Iterator[tuple[int, Output | Exception]]:
    """Run invoke in parallel, yielding results as they complete."""
```

**Yields:** Tuples of `(input_index, output)` in completion order, not input order.

**Use Case:** Process results as soon as available without waiting for all inputs to complete.

### Asynchronous Batch as Completed

**`abatch_as_completed(inputs, config=None, *, return_exceptions=False, **kwargs) → AsyncIterator[tuple[int, Output | Exception]]`** (`repo://libs/core/langchain_core/runnables/base.py#L1198-L1256`)

Async variant of `batch_as_completed`. Yields `(index, output)` tuples as `ainvoke` coroutines complete, respecting `max_concurrency` limits.

```python
async def abatch_as_completed(
    self,
    inputs: Sequence[Input],
    config: RunnableConfig | Sequence[RunnableConfig] | None = None,
    *,
    return_exceptions: bool = False,
    **kwargs: Any | None,
) -> AsyncIterator[tuple[int, Output | Exception]]:
    """Run ainvoke in parallel, yielding results as they complete."""
```

**Yields:** Tuples of `(input_index, output)` in completion order.

## Streaming Methods

### Synchronous Streaming

**`stream(input, config=None, **kwargs) → Iterator[Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L1258-L1277`)

Yields partial outputs as they are produced. Default implementation yields a single output from `invoke`.

```python
def stream(
    self,
    input: Input,
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> Iterator[Output]:
    """Yield outputs as they are produced."""
```

**Returns:** Iterator yielding outputs (typically chunks or intermediate results).

**Specialization:** LLMs and parsers override to yield tokens or partial results progressively. For example, a chat model might yield `AIMessageChunk` objects token-by-token.

**Transform Method:** Advanced streaming uses `transform(iterator)` to enable output while input is still being generated. Default `transform` buffers all input, then calls `stream`. Specialized implementations can yield from both input and output streams concurrently.

### Asynchronous Streaming

**`astream(input, config=None, **kwargs) → AsyncIterator[Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L1279-L1298`)

Async variant of `stream`. Default implementation yields a single output from `ainvoke`.

```python
async def astream(
    self,
    input: Input,
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> AsyncIterator[Output]:
    """Yield outputs asynchronously as they are produced."""
```

**Returns:** Async iterator yielding outputs.

### Transform: Iterator-to-Iterator Transformation

**`transform(input, config=None, **kwargs) → Iterator[Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L1824-L1861`)

Maps an iterator of inputs to an iterator of outputs. Default implementation buffers all input and calls `stream`. Specialized implementations can start producing output while input is still being generated.

```python
def transform(
    self,
    input: Iterator[Input],
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> Iterator[Output]:
    """Transform inputs to outputs."""
```

**Purpose:** Enables composing streaming chains where downstream runnables receive output chunks as upstream processing produces them, without waiting for all input.

**Override for:** Components that can start processing output before all input is available (e.g., incremental JSON parsing, line-by-line text processing).

### Async Transform

**`atransform(input, config=None, **kwargs) → AsyncIterator[Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L1863-L1901`)

Async variant of `transform`. Maps async iterator of inputs to async iterator of outputs.

```python
async def atransform(
    self,
    input: AsyncIterator[Input],
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> AsyncIterator[Output]:
    """Transform async inputs to async outputs."""
```

## Event Streaming (Advanced)

### Stream Events

**`astream_events(input, config=None, *, version='v2', include_names=None, include_types=None, include_tags=None, exclude_names=None, exclude_types=None, exclude_tags=None, **kwargs) → AsyncIterator[StreamEvent]`** (`repo://libs/core/langchain_core/runnables/base.py#L1432-L1735`)

Streams lifecycle events from the runnable and its child runnables. Events include start, stream, and end for each component in a chain.

```python
def astream_events(
    self,
    input: Any,
    config: RunnableConfig | None = None,
    *,
    version: Literal["v1", "v2", "v3"] = "v2",
    include_names: Sequence[str] | None = None,
    include_types: Sequence[str] | None = None,
    include_tags: Sequence[str] | None = None,
    exclude_names: Sequence[str] | None = None,
    exclude_types: Sequence[str] | None = None,
    exclude_tags: Sequence[str] | None = None,
    **kwargs: Any,
) -> AsyncIterator[StreamEvent]:
    """Generate a stream of events."""
```

**Parameters:**
- `version`: Event schema version:
  - `"v2"` (recommended): Yields `StreamEvent` dicts with support for custom events and parent tracking.
  - `"v1"`: Legacy event format; deprecated in favor of v2.
  - `"v3"`: Typed, content-block-centric streaming protocol; supported on `BaseChatModel` and `langgraph.CompiledGraph` only.
- `include_names/types/tags`: Filter events by runnable name, type, or tags (inclusion filter).
- `exclude_names/types/tags`: Exclude events matching criteria (exclusion filter).

**Event Format:** Dictionary with keys:
- `event`: Event name in format `"on_[runnable_type]_(start|stream|end)"`.
- `name`: Name of the runnable that generated the event.
- `run_id`: Unique identifier for this execution.
- `parent_ids`: IDs of parent runnables (v2 only).
- `tags`: Tags attached to the runnable.
- `metadata`: Metadata attached to the runnable.
- `data`: Dictionary with event data (`input`, `output`, `chunk`, `error` as applicable).

**Event Types:**
- `on_chain_start/stream/end`: Standard runnable lifecycle.
- `on_llm_start/stream/end`: Language model execution.
- `on_chat_model_start/stream/end`: Chat model execution.
- `on_prompt_start/end`: Prompt template execution.
- `on_tool_start/end`: Tool execution.
- `on_retriever_start/end`: Retriever execution.
- `on_custom_event`: Custom events dispatched via `adispatch_custom_event`.

**Use Case:** Real-time visibility into chain execution, intermediate results, token streaming, and error tracking.

### Sync Stream Events

**`stream_events(input, config=None, *, version='v2', ...) → Iterator[StreamEvent]`** (`repo://libs/core/langchain_core/runnables/base.py#L1763-L1822`)

Synchronous variant of `astream_events`. Base implementation raises `NotImplementedError` for v1/v2; subclasses may implement v3.

## Streaming Logs (Deprecated)

**`astream_log(input, config=None, *, diff=True, with_streamed_output_list=True, ...) → AsyncIterator[RunLogPatch] | AsyncIterator[RunLog]`** (`repo://libs/core/langchain_core/runnables/base.py#L1334-L1404`)

Streams output and selected intermediate results from an input. **Deprecated in favor of `astream_events` and `astream`.**

## Schema Introspection

### Type Properties

**`InputType: type`** (property, `repo://libs/core/langchain_core/runnables/base.py#L372-L405`)

The Python type annotation for accepted input. Extracted from generic parameters via `__orig_bases__` or Pydantic metadata.

```python
@property
def InputType(self) -> type[Input]:
    """Input type."""
```

**`OutputType: type`** (property, `repo://libs/core/langchain_core/runnables/base.py#L407-L436`)

The Python type annotation for produced output.

### Input Schema

**`input_schema: TypeBaseModel`** (property, `repo://libs/core/langchain_core/runnables/base.py#L438-L441`)

Returns a Pydantic model validating expected input shape. Derived from `InputType`.

```python
@property
def input_schema(self) -> TypeBaseModel:
    """The type of input this Runnable accepts specified as a Pydantic model."""
```

**Use:** Validation, documentation, tool interfaces, LangSmith introspection.

**`get_input_schema(config=None) → TypeBaseModel`** (`repo://libs/core/langchain_core/runnables/base.py#L443-L482`)

Get input schema for a specific config. Useful for runnables with configurable input schemas via `configurable_fields()` or `configurable_alternatives()`.

**`get_input_jsonschema(config=None) → dict`** (`repo://libs/core/langchain_core/runnables/base.py#L484-L512`)

Get JSON schema representation of input for use in OpenAPI/tool definitions.

### Output Schema

**`output_schema: TypeBaseModel`** (property, `repo://libs/core/langchain_core/runnables/base.py#L514-L520`)

Returns a Pydantic model validating expected output shape.

**`get_output_schema(config=None) → TypeBaseModel`** (`repo://libs/core/langchain_core/runnables/base.py#L522-L561`)

Get output schema for a specific config.

**`get_output_jsonschema(config=None) → dict`** (`repo://libs/core/langchain_core/runnables/base.py#L563-L591`)

Get JSON schema representation of output.

### Config Schema

**`config_schema(*, include=None) → type[BaseModel]`** (`repo://libs/core/langchain_core/runnables/base.py#L598-L639`)

Returns Pydantic model for configuration options accepted by this runnable. Includes configurable fields defined via `configurable_fields()` and `configurable_alternatives()`.

**`get_config_jsonschema(*, include=None) → dict`** (`repo://libs/core/langchain_core/runnables/base.py#L641-L655`)

Get JSON schema representation of config.

**`config_specs: list[ConfigurableFieldSpec]`** (property, `repo://libs/core/langchain_core/runnables/base.py#L593-L596`)

List of configurable field specs for runnables using `configurable_fields()` or `configurable_alternatives()`.

## Composition Operators

### Pipe Operator

**`__or__(other) → RunnableSequence`** (`repo://libs/core/langchain_core/runnables/base.py#L712-L737`)

Chains runnables sequentially: `runnable1 | runnable2 | runnable3`. Output of one becomes input to next.

```python
def __or__(
    self,
    other: Runnable[Output, Other] | Callable[[Output], Other] | Mapping[str, Any],
) -> RunnableSerializable[Input, Other]:
    """Runnable pipe operator. Compose this Runnable with another."""
```

**Accepts:**
- `Runnable` instances
- Callables (converted to `RunnableLambda`)
- Dict literals (converted to `RunnableParallel`)

**Returns:** `RunnableSequence` composing both runnables.

**Example:**
```python
chain = prompt | model | output_parser
result = chain.invoke({"input": "..."})
```

### Reverse Pipe Operator

**`__ror__(other) → RunnableSequence`** (`repo://libs/core/langchain_core/runnables/base.py#L761-L786`)

Right-associative pipe: `x | runnable` where `x` is not a `Runnable`. Enables composition with non-Runnable objects on the left.

### Pipe Method

**`pipe(*others, name=None) → RunnableSequence`** (`repo://libs/core/langchain_core/runnables/base.py#L788-L827`)

Explicit pipe method: `runnable.pipe(other1, other2, ...)` equivalent to `runnable | other1 | other2 | ...`.

## Configuration and Binding

### Bind Arguments

**`bind(**kwargs) → Runnable[Input, Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L1903-L1935`)

Binds keyword arguments to a runnable, returning a new runnable that invokes with those args bound. Used for passing parameters not in chain input.

```python
def bind(self, **kwargs: Any) -> Runnable[Input, Output]:
    """Bind arguments to a Runnable."""
```

**Use Case:** Pass parameters not in chain input (e.g., `model.bind(temperature=0.5, stop=["\\n"])`).

**Returns:** New `RunnableBinding` wrapping this runnable with bound kwargs.

**Example:**
```python
model = ChatOpenAI(model="gpt-4")
model_with_temp = model.bind(temperature=0)
```

### Bind Config

**`with_config(config=None, **kwargs) → Runnable[Input, Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L1937-L1960`)

Binds configuration (tags, metadata, callbacks, max_concurrency) to a runnable for tracing, observability, and execution control.

```python
def with_config(
    self,
    config: RunnableConfig | None = None,
    **kwargs: Any,
) -> Runnable[Input, Output]:
    """Bind config to a Runnable."""
```

**Use Case:** Add tracing tags, metadata, or callbacks to a chain segment.

**Returns:** New `RunnableBinding` with bound config.

**Example:**
```python
chain = (
    prompt
    | model.with_config({"tags": ["model"], "metadata": {"version": "v1"}})
    | parser
)
```

## Lifecycle and Error Handling

### Listeners

**`with_listeners(*, on_start=None, on_end=None, on_error=None) → Runnable[Input, Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L1962-L2032`)

Binds lifecycle callbacks (start, end, error) to a runnable for synchronous execution.

```python
def with_listeners(
    self,
    *,
    on_start: Callable[[Run], None] | Callable[[Run, RunnableConfig], None] | None = None,
    on_end: Callable[[Run], None] | Callable[[Run, RunnableConfig], None] | None = None,
    on_error: Callable[[Run], None] | Callable[[Run, RunnableConfig], None] | None = None,
) -> Runnable[Input, Output]:
    """Bind lifecycle listeners to a Runnable."""
```

**Parameters:**
- `on_start`: Called before execution with `Run` object. Signature: `(run: Run) → None` or `(run: Run, config: RunnableConfig) → None`.
- `on_end`: Called after successful execution with `Run` object.
- `on_error`: Called on exception with `Run` object.

**Returns:** New `RunnableBinding` with listener callbacks.

### Async Listeners

**`with_alisteners(*, on_start=None, on_end=None, on_error=None) → Runnable[Input, Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L2034-L2135`)

Async variant of `with_listeners` for use in async execution contexts. Supports both sync and async callback signatures.

### Retry Policy

**`with_retry(stop_after_attempt=3, wait_exponential_jitter=True, retry_if_exception_type=(Exception,), exponential_jitter_params=None) → Runnable[Input, Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L2153-L2218`)

Binds automatic retry logic to a runnable with exponential backoff and jitter.

```python
def with_retry(
    self,
    *,
    stop_after_attempt: int = 3,
    wait_exponential_jitter: bool = True,
    retry_if_exception_type: tuple[type[BaseException], ...] = (Exception,),
    exponential_jitter_params: ExponentialJitterParams | None = None,
) -> Runnable[Input, Output]:
    """Bind a retry policy to the underlying Runnable."""
```

**Parameters:**
- `stop_after_attempt`: Max retry attempts (default: 3).
- `wait_exponential_jitter`: Use exponential backoff with jitter (default: True).
- `retry_if_exception_type`: Tuple of exception types to retry on (default: all exceptions).
- `exponential_jitter_params`: Custom backoff parameters.

**Returns:** New `RunnableRetry` wrapper.

**Example:**
```python
model = ChatOpenAI().with_retry(
    stop_after_attempt=5,
    wait_exponential_jitter=True,
    retry_if_exception_type=(TimeoutError, RateLimitError)
)
```

### Fallbacks

**`with_fallbacks(fallbacks, *, exceptions_to_handle=(Exception,), exception_key=None) → RunnableWithFallbacks[Input, Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L2243-L2319`)

Chains fallback runnables to try on failure. Implements graceful degradation.

```python
def with_fallbacks(
    self,
    fallbacks: Sequence[Runnable[Input, Output]],
    *,
    exceptions_to_handle: tuple[type[BaseException], ...] = (Exception,),
    exception_key: str | None = None,
) -> RunnableWithFallbacksT[Input, Output]:
    """Add fallbacks to a Runnable."""
```

**Parameters:**
- `fallbacks`: Sequence of runnables to try in order on failure.
- `exceptions_to_handle`: Exception types to handle (default: all exceptions).
- `exception_key`: If set (string), exception is passed to fallbacks under this key in input dict.

**Use Case:** Try primary method, fall back to alternative on error (e.g., primary LLM → backup LLM; online retrieval → fallback index).

**Returns:** New `RunnableWithFallbacks` wrapper.

**Example:**
```python
chain = (
    primary_model
    .with_fallbacks([backup_model1, backup_model2])
    | parser
)
```

### Map

**`map() → Runnable[Sequence[Input], list[Output]]`** (`repo://libs/core/langchain_core/runnables/base.py#L2220-L2241`)

Returns a runnable that applies this runnable to each element of an input sequence.

```python
def map(self) -> Runnable[Sequence[Input], list[Output]]:
    """Return a new Runnable that maps a list of inputs to list of outputs."""
```

**Equivalent to:** Calling `invoke` on each element and collecting results.

**Returns:** New `RunnableEach` wrapper.

**Example:**
```python
from langchain_core.runnables import RunnableLambda

square = RunnableLambda(lambda x: x ** 2)
mapped = square.map()
mapped.invoke([1, 2, 3])  # [1, 4, 9]
```

## Utility Methods

### Get Name

**`get_name(suffix=None, *, name=None) → str`** (`repo://libs/core/langchain_core/runnables/base.py#L334-L370`)

Retrieve the runnable's name, used for tracing and debugging. Extracted from `self.name`, class name, or custom override.

```python
def get_name(self, suffix: str | None = None, *, name: str | None = None) -> str:
    """Get the name of the Runnable."""
```

**Parameters:**
- `suffix`: Optional suffix appended to name (e.g., `"Input"`, `"Output"`).
- `name`: Override with custom name.

**Returns:** String name of runnable.

### Get Graph

**`get_graph(config=None) → Graph`** (`repo://libs/core/langchain_core/runnables/base.py#L657-L676`)

Returns a graph representation of the runnable's structure.

**Use Case:** Visualization, debugging complex compositions.

### Get Prompts

**`get_prompts(config=None) → list[BasePromptTemplate]`** (`repo://libs/core/langchain_core/runnables/base.py#L678-L689`)

Extracts all prompt templates used by this runnable and its children recursively.

**Use Case:** Introspect prompts in a chain; useful for prompt versioning and analysis.

## Configuration Management

### RunnableConfig

`RunnableConfig` is a `TypedDict` that carries execution context through a chain. All keys are optional (`total=False`).

```python
class RunnableConfig(TypedDict, total=False):
    """Configuration for a Runnable."""
    tags: list[str]
    metadata: dict[str, Any]
    callbacks: Callbacks
    run_name: str
    max_concurrency: int | None
    recursion_limit: int
    configurable: dict[str, Any]
    run_id: uuid.UUID | None
```

**Fields:**

- **`tags`**: List of strings for tracing and filtering. Inherited by child runnables.
- **`metadata`**: Dictionary of JSON-serializable metadata. Inherited by child runnables.
- **`callbacks`**: List of callback handlers. Inherited by child runnables.
- **`run_name`**: Name for tracer run (defaults to runnable class name).
- **`max_concurrency`**: Limits parallel operations in `batch`, `abatch`, and their `*_as_completed` variants.
- **`recursion_limit`**: Maximum recursion depth (default: 25).
- **`configurable`**: Runtime values for attributes made configurable via `configurable_fields()` or `configurable_alternatives()`.
- **`run_id`**: Unique UUID for tracer run (auto-generated if not provided).

**Config Threading:**

Configs flow from parent to child runnables automatically via context variables (`var_child_runnable_config`). Child configs are merged with parent configs using `merge_configs()` and `ensure_config()`, with child values overriding parent values where specified.

### ensure_config

**`ensure_config(config: RunnableConfig | None = None) → RunnableConfig`** (`repo://libs/core/langchain_core/runnables/config.py#L255-L308`)

Ensures a config is a complete dict with all keys present, using defaults and merging with parent config from context.

```python
def ensure_config(config: RunnableConfig | None = None) -> RunnableConfig:
    """Ensure that a config is a dict with all keys present."""
```

**Behavior:**
1. Starts with empty dict containing defaults (empty tags/metadata, recursion_limit=25).
2. Merges parent config from `var_child_runnable_config` context variable if present.
3. Merges provided config if not None.
4. Non-standard config keys go into `configurable` dict.
5. Automatically infers metadata for `model` and `checkpoint_ns` keys from configurable.

**Use:** Called internally by all execution methods to prepare config before use.

### get_config_list

**`get_config_list(config: RunnableConfig | Sequence[RunnableConfig] | None, length: int) → list[RunnableConfig]`** (`repo://libs/core/langchain_core/runnables/config.py#L311-L363`)

Converts a single config or list of configs into a list of ensured configs matching the given length.

```python
def get_config_list(
    config: RunnableConfig | Sequence[RunnableConfig] | None, length: int
) -> list[RunnableConfig]:
    """Get a list of configs from a single config or a list of configs."""
```

**Behavior:**
- If `config` is None: returns list of `length` default configs via `ensure_config()`.
- If `config` is a single dict: returns list of `length` copies, optionally dropping `run_id` for items after the first.
- If `config` is a sequence: validates length matches and returns ensured list.

**Use:** Called by `batch`, `abatch`, and related methods to expand config for each input.

## Helper Methods for Subclasses

Subclasses override abstract `invoke` and typically implement execution logic using these helper methods that handle config, callbacks, and error management:

**`_call_with_config(func, input_, config, run_type=None, serialized=None, **kwargs) → Output`** (`repo://libs/core/langchain_core/runnables/base.py#L2323-L2370`)

Wraps a callable with config management, callbacks, and error handling. Use in `invoke` implementations.

**`_acall_with_config(func, input_, config, run_type=None, serialized=None, **kwargs) → Coroutine[Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L2372-L2413`)

Async variant for `ainvoke` implementations.

**`_batch_with_config(func, inputs, config, *, return_exceptions=False, run_type=None, **kwargs) → list[Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L2415-L2480`)

Wraps batch processing function with callback management.

**`_abatch_with_config(func, inputs, config, *, return_exceptions=False, run_type=None, **kwargs) → Coroutine[list[Output]]`** (`repo://libs/core/langchain_core/runnables/base.py#L2482-L2555`)

Async variant for `abatch` implementations.

**`_transform_stream_with_config(inputs, transformer, config, run_type=None, **kwargs) → Iterator[Output]`** (`repo://libs/core/langchain_core/runnables/base.py#L2557-L2630`)

Wraps streaming transformer function with callback lifecycle.

## Composition Primitives (LCEL)

The `Runnable` protocol supports composable primitives for building complex chains:

**Sequential (RunnableSequence):** Chain with `|` operator. Output of each becomes input to next.

```python
chain = prompt | model | output_parser
result = chain.invoke({"input": "..."})
```

**Parallel (RunnableParallel):** Use dict literal `{ "key1": runnable1, "key2": runnable2 }` to invoke runnables concurrently with shared input.

```python
branch = {
    "summary": model | StrOutputParser(),
    "entities": entity_extractor,
}
result = branch.invoke(document)  # {"summary": "...", "entities": [...]}
```

**Branching:** Conditional routing using mapping or custom logic.

**Fallback:** Try primary, fall back to secondary on failure (see `with_fallbacks`).

**Retry:** Automatic exponential backoff retry (see `with_retry`).

All composed chains inherit full sync, async, batch, and streaming support automatically from the protocol.

## Implementation Patterns

**Minimal Runnable:** Override `invoke` only; all other methods work automatically with defaults.

```python
from langchain_core.runnables import Runnable, RunnableConfig

class MyRunnable(Runnable[str, str]):
    def invoke(
        self,
        input: str,
        config: RunnableConfig | None = None,
        **kwargs: Any,
    ) -> str:
        return input.upper()

runnable = MyRunnable()
runnable.invoke("hello")  # "HELLO"
runnable.batch(["hello", "world"])  # ["HELLO", "WORLD"]
runnable.stream("hello")  # yields "HELLO"
```

**Optimized Runnable:** Override methods for efficiency (native async, batch API support, streaming).

```python
class OptimizedRunnable(Runnable[list[str], list[str]]):
    async def ainvoke(
        self,
        input: list[str],
        config: RunnableConfig | None = None,
        **kwargs: Any,
    ) -> list[str]:
        # Native async instead of executor
        return [x.upper() for x in input]
    
    def batch(
        self,
        inputs: list[list[str]],
        config: RunnableConfig | list[RunnableConfig] | None = None,
        *,
        return_exceptions: bool = False,
        **kwargs: Any,
    ) -> list[list[str]]:
        # Optimized batch processing
        return [
            [item.upper() for item in input_list]
            for input_list in inputs
        ]
    
    def stream(
        self,
        input: list[str],
        config: RunnableConfig | None = None,
        **kwargs: Any,
    ) -> Iterator[str]:
        # Stream results incrementally
        for item in input:
            yield item.upper()
```

**Streaming Runnable:** Override `stream` to yield chunks or partial results.

```python
class StreamingRunnable(Runnable[str, str]):
    def invoke(self, input: str, config: RunnableConfig | None = None, **kwargs: Any) -> str:
        return input.upper()
    
    def stream(self, input: str, config: RunnableConfig | None = None, **kwargs: Any) -> Iterator[str]:
        # Yield characters one at a time
        result = input.upper()
        for char in result:
            yield char
```
