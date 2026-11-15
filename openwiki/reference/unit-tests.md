---
type: API Reference
title: Unit Tests Reference
description: API reference for pytest fixtures, mock implementations, and test utilities provided by LangChain's test infrastructure.
tags: [unit-tests, pytest, fixtures, testing, reference, api]
verified:
  - by: openwiki/0.5.0
    at: 2026-11-15T08:15:26.682Z
sources:
  - id: openwiki-source-727aef6a92fb635fdbb41cd6
    resource: repo://libs/core/tests/unit_tests/conftest.py
  - id: openwiki-source-5e13d2c899eb5925ef28fddf
    resource: repo://libs/core/tests/unit_tests/fake/callbacks.py
  - id: openwiki-source-e8916b46b41eee662deabd17
    resource: repo://libs/core/tests/unit_tests/fake/test_fake_chat_model.py
  - id: openwiki-source-f0e376fe9b6befdcc2505465
    resource: repo://libs/core/tests/unit_tests/pydantic_utils.py
  - id: openwiki-source-5839db669f618a6d604790ca
    resource: repo://libs/core/tests/unit_tests/stubs.py
  - id: openwiki-source-09fd21baed875d7f9c53a492
    resource: repo://libs/langchain_v1/tests/unit_tests/agents/conftest.py
  - id: openwiki-source-6904a2d923d0abd0c1af93bd
    resource: repo://libs/langchain_v1/tests/unit_tests/agents/model.py
generated: { by: "openwiki/0.5.0", at: "2026-11-15T08:15:26.682Z" }
---

## Overview

This page documents the fixtures, helper functions, and mock implementations available in LangChain's unit test infrastructure. These utilities enable consistent, network-independent testing across the codebase without API calls or external dependencies.

## Core Test Configuration (conftest.py)

### blockbuster Fixture

**Location**: `repo://libs/core/tests/unit_tests/conftest.py`

The `blockbuster` fixture (autouse=True) prevents blocking I/O in async code by permitting specific blocking operations only in designated locations. This enforces non-blocking patterns in async functions.

```python
@pytest.fixture(autouse=True)
def blockbuster() -> Iterator[BlockBuster]:
    """Prevents blocking I/O in async code."""
    with blockbuster_ctx("langchain_core") as bb:
        for func in ["os.stat", "os.path.abspath"]:
            (
                bb.functions[func]
                .can_block_in("langchain_core/_api/internal.py", "is_caller_internal")
                .can_block_in("langchain_core/runnables/base.py", "__repr__")
            )
        for func in ["os.stat", "io.TextIOWrapper.read"]:
            bb.functions[func].can_block_in(
                "langsmith/client.py", "_default_retry_config"
            )
        for bb_function in bb.functions.values():
            bb_function.can_block_in(
                "freezegun/api.py", "_get_cached_module_attributes"
            )
        yield bb
```

**Usage**: Automatically applied to all tests. If a test hits blocking I/O outside permitted locations, it fails with a clear error indicating the function and location.

### deterministic_uuids Fixture

**Location**: `repo://libs/core/tests/unit_tests/conftest.py`

Replaces random UUID generation with deterministic, sequential values for reproducible snapshot testing.

```python
@pytest.fixture
def deterministic_uuids(mocker: MockerFixture) -> MockerFixture:
    side_effect = (
        UUID(f"00000000-0000-4000-8000-{i:012}", version=4) for i in range(10000)
    )
    return mocker.patch("uuid.uuid4", side_effect=side_effect)
```

**Usage**: Use in tests where UUIDs must be stable across runs (e.g., trace snapshots, message IDs):

```python
def test_trace_snapshot(deterministic_uuids, snapshot):
    # UUIDs are now predictable: 00000000-0000-4000-8000-000000000000, ...
    result = create_trace()
    assert result == snapshot
```

### pytest_collection_modifyitems Hook

**Location**: `repo://libs/core/tests/unit_tests/conftest.py`

Implements custom `@pytest.mark.requires` marker for conditional test execution based on optional dependencies.

**`@pytest.mark.requires(package_name)`**: Auto-skips tests when packages are missing; fails if `--only-extended` is used and a required package is not installed.

```python
@pytest.mark.requires("anthropic", "openai")
def test_multi_provider():
    # Test skipped if anthropic or openai not installed
    pass
```

**Command-line options**:
- `--only-extended`: Run only tests marked with `@pytest.mark.requires`; fail if any required dependencies are missing
- `--only-core`: Run only non-extended tests (skip all `@pytest.mark.requires`)

### pytest_addoption Hook

**Location**: `repo://libs/core/tests/unit_tests/conftest.py`

Adds custom command-line options:

```python
def pytest_addoption(parser: pytest.Parser) -> None:
    """Add custom command line options to pytest."""
    parser.addoption(
        "--only-extended",
        action="store_true",
        help="Only run extended tests. Does not allow skipping any extended tests.",
    )
    parser.addoption(
        "--only-core",
        action="store_true",
        help="Only run core tests. Never runs any extended tests.",
    )
```

## Checkpoint and Store Fixtures (Agent Testing)

**Location**: `repo://libs/langchain_v1/tests/unit_tests/agents/conftest.py`

Agent tests use parametrized fixtures to test across multiple storage backends. Control backend selection via `LANGGRAPH_TEST_FAST` environment variable (default: `true`).

### sync_checkpointer Fixture

Parametrized fixture yielding different checkpoint storage backends.

```python
@pytest.fixture(params=SYNC_CHECKPOINTER_PARAMS)
def sync_checkpointer(request: pytest.FixtureRequest) -> Iterator[BaseCheckpointSaver[str]]:
    """Provides checkpointer for multiple backends."""
    checkpointer_name = request.param
    if checkpointer_name == "memory":
        with _checkpointer_memory() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "sqlite":
        with _checkpointer_sqlite() as checkpointer:
            yield checkpointer
    # ... more backends
```

**Available backends**:
- `memory` (always available)
- `sqlite`, `postgres`, `postgres_pipe`, `postgres_pool` (when `LANGGRAPH_TEST_FAST=false`)

**Usage**:

```python
def test_agent_checkpoint(sync_checkpointer):
    """Test runs once per backend."""
    agent = create_agent(model, [tool], checkpointer=sync_checkpointer)
    config = {"configurable": {"thread_id": "t1"}}
    result = agent.invoke({"messages": [HumanMessage("hi")]}, config)
```

### async_checkpointer Fixture

Async variant for async agent tests.

```python
@pytest.fixture(params=ASYNC_CHECKPOINTER_PARAMS)
async def async_checkpointer(request: pytest.FixtureRequest) -> AsyncIterator[BaseCheckpointSaver[str]]:
    """Provides checkpointer for multiple backends (async)."""
```

**Available backends**:
- `memory` (always available)
- `sqlite_aio`, `postgres_aio`, `postgres_aio_pipe`, `postgres_aio_pool` (when fast mode disabled)

### sync_store & async_store Fixtures

**Location**: `repo://libs/langchain_v1/tests/unit_tests/agents/conftest.py`

Parametrized fixtures for testing with different storage implementations.

```python
@pytest.fixture(params=SYNC_STORE_PARAMS)
def sync_store(request: pytest.FixtureRequest) -> Iterator[BaseStore | None]:
    """Provides store for multiple backends."""
```

**Available backends**:
- `in_memory` (always available)
- `postgres`, `postgres_pipe`, `postgres_pool` (when fast mode disabled)

### anyio_backend Fixture

**Location**: `repo://libs/langchain_v1/tests/unit_tests/agents/conftest.py`

Configures asyncio as the backend for async tests (pytest-anyio integration).

```python
@pytest.fixture
def anyio_backend() -> str:
    return "asyncio"
```

## Fake/Mock Chat Models

**Location**: `repo://libs/core/langchain_core/language_models/fake_chat_models.py`

Mock implementations enable testing without API calls.

### FakeChatModel

The simplest fake chat model that returns a fixed response.

```python
class FakeChatModel(SimpleChatModel):
    """Fake Chat Model wrapper for testing purposes."""
    
    def _call(self, messages: list[BaseMessage], ...) -> str:
        return "fake response"
```

**Usage**:

```python
model = FakeChatModel()
result = model.invoke([HumanMessage("Hello")])
# Returns AIMessage(content="fake response")
```

### FakeListChatModel

Cycles through a fixed list of string responses.

```python
class FakeListChatModel(SimpleChatModel):
    responses: list[str]
    """List of responses to **cycle** through in order."""
    sleep: float | None = None
    """Optional sleep time in seconds between responses."""
    i: int = 0
    """Internally incremented after every model invocation."""
    error_on_chunk_number: int | None = None
    """If set, raise an error on the specified chunk number during streaming."""
```

**Usage**:

```python
model = FakeListChatModel(responses=["Hello", "Hi", "Hey"])
response = model.invoke("How are you?")
# Returns AIMessage(content="Hello")

response = model.invoke("What's up?")
# Returns AIMessage(content="Hi")

# Streaming
chunks = list(model.stream("Query"))
# Returns character-level chunks
```

### GenericFakeChatModel

Cycles through a sequence of AIMessage objects; useful for testing streaming with complex messages.

```python
class GenericFakeChatModel(BaseChatModel):
    messages: Iterator[AIMessage | str]
    """Get an iterator over messages."""
    
    def _stream(self, messages: list[BaseMessage], ...) -> Iterator[ChatGenerationChunk]:
        """Breaks messages into character-level chunks for streaming tests."""
```

**Usage**:

```python
from itertools import cycle
from langchain_core.language_models import GenericFakeChatModel
from langchain_core.messages import AIMessage

messages = cycle([AIMessage(content="hello"), AIMessage(content="goodbye")])
model = GenericFakeChatModel(messages=messages)

# Invoke returns first message
response = model.invoke("query")
# Returns AIMessage(content="hello")

# Stream returns character-level chunks
chunks = list(model.stream("query"))
# Returns [AIMessageChunk(content="h"), AIMessageChunk(content="e"), ...]
```

### ParrotFakeChatModel

Echoes the input message (last message in the conversation) back as an AIMessage.

```python
class ParrotFakeChatModel(BaseChatModel):
    """Returns the last message from input unchanged."""
    
    def _generate(self, messages: list[BaseMessage], ...) -> ChatResult:
        return ChatResult(generations=[ChatGeneration(message=messages[-1])])
```

**Usage**:

```python
from langchain_core.language_models import ParrotFakeChatModel

model = ParrotFakeChatModel()
response = model.invoke([HumanMessage("Hello!")])
# Returns HumanMessage(content="Hello!")
```

### FakeMessagesListChatModel

Cycles through a fixed list of BaseMessage objects (not just strings).

```python
class FakeMessagesListChatModel(BaseChatModel):
    responses: list[BaseMessage]
    """List of responses to **cycle** through in order."""
    sleep: float | None = None
    """Optional sleep time in seconds between responses."""
    i: int = 0
```

### FakeToolCallingModel

**Location**: `repo://libs/langchain_v1/tests/unit_tests/agents/model.py`

Mock chat model that returns scripted tool calls for testing agent behavior across multiple turns.

```python
class FakeToolCallingModel(BaseChatModel):
    tool_calls: list[list[ToolCall]] | list[list[dict[str, Any]]] | None = None
    structured_response: Any | None = None
    index: int = 0
    tool_style: Literal["openai", "anthropic"] = "openai"
    
    def _generate(self, messages: list[BaseMessage], ...) -> ChatResult:
        """Returns scripted tool calls or structured responses."""
```

**Parameters**:

- `tool_calls`: List of lists, where each inner list contains tool calls for that turn. Cycles or increments based on native mode.
- `structured_response`: Response for structured output mode (BaseModel, dataclass, or dict).
- `index`: Internal counter; auto-increments after each invoke.
- `tool_style`: Format for bind_tools output ("openai" or "anthropic").

**Usage**:

```python
# Multi-turn tool calling sequence
model = FakeToolCallingModel(
    tool_calls=[
        [ToolCall(name="get_weather", args={"city": "Paris"}, id="call_1")],
        [],  # No tool call on second turn (agent finishes)
    ]
)

# With structured output
from pydantic import BaseModel

class WeatherResponse(BaseModel):
    city: str
    temperature: int

model = FakeToolCallingModel(
    structured_response=WeatherResponse(city="Paris", temperature=20)
)
result = model.invoke([...], response_format=...)
```

### FakeEmbeddings

Generates deterministic embeddings based on input hash.

```python
from langchain_core.embeddings import FakeEmbeddings

embeddings = FakeEmbeddings(model="fake-model", size=1536)

# Embed a single string
vector = embeddings.embed_query("hello")
# Returns list of 1536 float values (deterministic)

# Embed a batch
vectors = embeddings.embed_documents(["hello", "world"])
# Returns list of vectors, one per input
```

## Callback Test Utilities

**Location**: `repo://libs/core/tests/unit_tests/fake/callbacks.py`

### FakeCallbackHandler

Tracks callback events for testing. Maintains counters for all lifecycle events at fine granularity.

```python
class FakeCallbackHandler(BaseCallbackHandler, BaseFakeCallbackHandlerMixin):
    """Fake callback handler for testing."""
    
    # Aggregate counters
    starts: int = 0
    ends: int = 0
    errors: int = 0
    text: int = 0
    
    # Fine-grained counters
    llm_starts: int = 0
    llm_ends: int = 0
    llm_streams: int = 0
    chain_starts: int = 0
    chain_ends: int = 0
    tool_starts: int = 0
    tool_ends: int = 0
    agent_actions: int = 0
    agent_ends: int = 0
    chat_model_starts: int = 0
    retriever_starts: int = 0
    retriever_ends: int = 0
    retriever_errors: int = 0
    retries: int = 0
    
    # Configuration
    ignore_llm_: bool = False
    ignore_chain_: bool = False
    ignore_agent_: bool = False
    ignore_retriever_: bool = False
    ignore_chat_model_: bool = False
    fake_id: str | None = None
    errors_args: list[Any] = []
```

**Usage**:

```python
from tests.unit_tests.fake.callbacks import FakeCallbackHandler

handler = FakeCallbackHandler()

# Invoke with callbacks
model.invoke("query", callbacks=[handler])

# Assert callback events at fine granularity
assert handler.llm_starts == 1
assert handler.llm_ends == 1
assert handler.chain_starts == 0
assert handler.tool_starts == 0
```

**Event Properties**:

- `ignore_llm`, `ignore_chain`, `ignore_agent`, `ignore_retriever`, `ignore_chat_model`: Skip tracking for specific component types
- `errors_args`: Captured arguments from error callbacks for detailed inspection
- `fake_id`: Optional identifier for comparing handlers (allows similar handlers to differ in equality)

## Message Testing Utilities

**Location**: `repo://libs/core/tests/unit_tests/stubs.py`

### AnyStr Helper

String subclass that matches any string value in equality comparisons. Useful for testing message IDs that are generated randomly.

```python
class AnyStr(str):
    """String that equals any other string."""
    
    def __eq__(self, other: object) -> bool:
        return isinstance(other, str)
```

**Usage**:

```python
from tests.unit_tests.stubs import AnyStr

any_str = AnyStr()
assert any_str == "any_string_value"  # True
assert any_str == "different_value"   # True
```

### Message Factory Functions

Create messages with AnyStr IDs for content-based assertions independent of generated UUIDs.

```python
def _any_id_ai_message(**kwargs: Any) -> AIMessage:
    """Create an `AIMessage` with an any id field."""
    message = AIMessage(**kwargs)
    message.id = AnyStr()
    return message

def _any_id_ai_message_chunk(**kwargs: Any) -> AIMessageChunk:
    """Create an `AIMessageChunk` with an any id field."""
    message = AIMessageChunk(**kwargs)
    message.id = AnyStr()
    return message

def _any_id_human_message(**kwargs: Any) -> HumanMessage:
    """Create a `HumanMessage` with an any id field."""
    message = HumanMessage(**kwargs)
    message.id = AnyStr()
    return message

def _any_id_document(**kwargs: Any) -> Document:
    """Create a `Document` with an id field."""
    message = Document(**kwargs)
    message.id = AnyStr()
    return message
```

**Usage**:

```python
from tests.unit_tests.stubs import (
    _any_id_ai_message,
    _any_id_ai_message_chunk,
    _any_id_human_message,
    _any_id_document,
)

# Create messages with wildcard IDs
ai_msg = _any_id_ai_message(content="hello")
human_msg = _any_id_human_message(content="hi")
doc = _any_id_document(page_content="text")

# Usage in assertions - matches content, ignores ID differences
response = model.invoke("query")
assert response == ai_msg
```

## Pydantic Schema Utilities

**Location**: `repo://libs/core/tests/unit_tests/pydantic_utils.py`

Utilities for schema normalization across Pydantic v1 and v2.

### skip_if_no_pydantic_v1

Marker to skip tests on Python 3.14+ where pydantic.v1 namespace is unavailable.

```python
from tests.unit_tests.pydantic_utils import skip_if_no_pydantic_v1

@skip_if_no_pydantic_v1
def test_pydantic_v1_model():
    from pydantic.v1 import BaseModel
    # Test only runs on Python < 3.14
```

### _schema Function

Extracts and normalizes JSON schemas from Pydantic models, handling differences between Pydantic v1 and v2.

```python
from tests.unit_tests.pydantic_utils import _schema
from pydantic import BaseModel

class MyModel(BaseModel):
    name: str
    age: int

schema = _schema(MyModel)
# Returns normalized JSON schema with v1/v2 differences resolved
```

### Schema Normalization Functions

Internal utilities for normalizing schemas across Pydantic versions:

- `replace_all_of_with_ref()`: Collapses single-element allOf into $ref
- `remove_all_none_default()`: Removes incorrect None defaults from v2 schemas (which uses NotRequired)
- `_remove_enum()`: Removes enum descriptions
- `_remove_additionalproperties()`: Removes additionalProperties: true from dict properties

## Environment and Mode Configuration

### LANGGRAPH_TEST_FAST

**Type**: Environment variable (boolean, default: `true`)

Controls which checkpoint and store backends are tested. Set to `true` for fast iteration with only in-memory backends, or `false` to test against all backends including sqlite and postgres.

```bash
# Fast mode: only in-memory
LANGGRAPH_TEST_FAST=true pytest tests/

# Full mode: all backends
LANGGRAPH_TEST_FAST=false pytest tests/
```

**Behavior**:
- When `true`: `sync_checkpointer` and `async_checkpointer` parametrize only over memory backends
- When `false`: Test against sqlite, postgres with various connection strategies (pipe, pool)

### hermetic_env Fixture (OpenAI Partner Tests)

**Location**: `repo://libs/partners/openai/tests/unit_tests/conftest.py`

Isolates unit tests from ambient OpenAI and gateway configuration by clearing all gateway and provider environment variables.

```python
@pytest.fixture(autouse=True)
def hermetic_env(monkeypatch: pytest.MonkeyPatch) -> None:
    """Isolate unit tests from ambient OpenAI and gateway configuration."""
    for var in (*_GATEWAY_VARS, *_PROVIDER_VARS):
        monkeypatch.delenv(var, raising=False)
    monkeypatch.setenv("OPENAI_API_KEY", "foo")
```

**Usage**: Automatically applied to all OpenAI partner tests. Ensures tests run offline against mocked clients without being affected by real credentials or base URLs.

## Test Infrastructure Patterns

### Running Tests

From the codebase Makefiles:

```bash
# Run all unit tests in parallel with socket restrictions
make test

# Run specific test file
make test TEST_FILE=tests/unit_tests/runnables/test_runnable.py

# Watch mode (auto-reruns and updates snapshots)
make test_watch

# Extended tests (requires optional dependencies)
make extended_tests
```

### Snapshot Testing

Tests use Syrupy for snapshot testing. Snapshots stored in `__snapshots__/*.ambr`.

```bash
# Update snapshots
pytest --snapshot-update

# Or via make
make test_watch  # Auto-updates on file changes
```

### Parametrization Pattern

```python
@pytest.mark.parametrize("input,expected", [
    ("hello", 5),
    ("world", 5),
])
def test_length(input, expected):
    assert len(input) == expected

# Parametrize with fixtures
@pytest.fixture(params=["memory", "sqlite"])
def backend(request):
    return request.param

def test_checkpoint(sync_checkpointer, backend):
    # Test runs once per backend
    pass
```

### Async Test Pattern

```python
# Using pytest-anyio with anyio_backend fixture
async def test_with_anyio():
    # Runs with asyncio backend
    model = GenericFakeChatModel(messages=iter([AIMessage(content="test")]))
    result = await model.ainvoke("Hello")
    assert result.content is not None
```

## Standard Test Classes

**Location**: `repo://libs/standard-tests/langchain_tests/`

Standard base test classes enforce consistent testing across integrations:

- `ChatModelUnitTests`: Base class for chat model implementations
- `EmbeddingsUnitTests`: Base class for embeddings implementations
- `ToolsUnitTests`: Base class for tool implementations

See `/openwiki/testing/unit-tests.md` for detailed documentation on standard test class configuration and supported feature flags.
