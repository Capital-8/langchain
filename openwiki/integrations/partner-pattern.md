---
type: "Integration Pattern"
title: "Partner Pattern: Provider Integration Framework"
description: "Architecture and implementation patterns for adding new provider integrations to LangChain. Covers ChatModel subclassing, message conversion, tool calling, and structured output support."
tags: [partner, integration, chat-model, tool-calling, structured-output, message-conversion, package-structure]
sources:
  - id: openwiki-source-c52037e7b642f7ac5a7642a8
    resource: repo://libs/core/langchain_core/language_models/chat_models.py
  - id: openwiki-source-a12395e96060f062d8ec1ca2
    resource: repo://libs/partners/anthropic/langchain_anthropic/__init__.py
  - id: openwiki-source-7de1ace618efbdfd8bacb5cb
    resource: repo://libs/partners/anthropic/langchain_anthropic/chat_models.py
  - id: openwiki-source-04e3ac4f56ff2adb2b02de7d
    resource: repo://libs/partners/anthropic/pyproject.toml
  - id: openwiki-source-1e66a9da38565f8901e651f4
    resource: repo://libs/partners/openai/langchain_openai/__init__.py
  - id: openwiki-source-738512768ef81ae009b097ac
    resource: repo://libs/partners/openai/langchain_openai/chat_models/base.py
  - id: openwiki-source-df762860acfcc6abf0ce804b
    resource: repo://libs/partners/openai/pyproject.toml
  - id: openwiki-source-86b6689572ac828885d7d4b0
    resource: repo://libs/partners/README.md
generated: { by: "openwiki/0.5.0", at: "2026-11-15T08:15:26.682Z" }
verified:
  - by: openwiki/0.5.0
    at: 2026-11-15T08:15:26.682Z
---

# Partner Pattern: Provider Integration Framework

LangChain's partner pattern defines how provider-specific integrations (like OpenAI, Anthropic, Ollama) implement support for chat models, embeddings, and other abstractions. Partner packages live in `libs/partners/<provider>/` and implement the core abstractions defined in `langchain-core` by adapting each provider's API to LangChain's unified message and runnable interfaces.

## Partner Package Structure

Each partner package follows a standardized layout. The OpenAI partner (`langchain-openai`) exemplifies this structure:

```
libs/partners/openai/
  langchain_openai/
    __init__.py              # Public API exports
    chat_models/
      base.py               # BaseChatOpenAI and ChatOpenAI implementations
      azure.py              # Azure-specific variant
      _client_utils.py      # Helper functions for client setup
      _compat.py            # Version/compatibility handling
    embeddings/
      base.py               # OpenAI embeddings integration
    llms/                   # Legacy completion API support
    tools/                  # Tool-specific helpers
    output_parsers/         # Custom output parsing logic
    data/
      _profiles.py          # Model capability profiles
  pyproject.toml            # Package metadata and dependencies
  README.md                 # Integration documentation
  Makefile                  # Development targets
  tests/                    # Test suite
```

### pyproject.toml Dependencies

Partner packages declare dependencies on:
- **`langchain-core`**: Provides base abstractions (`BaseChatModel`, `BaseRunnable`, messages, tools).
- **Provider SDK**: The official client library (e.g., `openai>=2.45.0`, `anthropic>=...`).
- **Optional extras**: For advanced features (e.g., `azure-identity` for Azure OpenAI).

Example (from langchain-openai):

```toml
[project]
name = "langchain-openai"
dependencies = [
    "langchain-core>=1.8.0,<2.0.0",
    "certifi>=2024.6.2",
    "openai>=2.45.0",
    "tiktoken>=0.7.0",
]

[project.optional-dependencies]
azure-identity = [
    "azure-identity>=1.25.0",
    "aiohttp>=3.11.0",
]
```

### Public API (_\_init\_\_.py)

Partners export a curated set of classes, typically:
- `ChatOpenAI`, `AzureChatOpenAI`: Chat model implementations
- `OpenAI`, `AzureOpenAI`: Legacy completion models (if supported)
- `OpenAIEmbeddings`: Embeddings integration
- Utility classes: `OpenAIDecisions`, `custom_tool`

Only stable, user-facing classes should be exported; internal helpers remain private.

## ChatModel Subclassing

The central pattern: every partner implements one or more **`BaseChatModel` subclasses** that wrap the provider's API. `ChatOpenAI` serves as the canonical example.

### Required Method Implementations

All `BaseChatModel` subclasses **must** implement:

#### `_generate(messages, stop, run_manager, **kwargs) → ChatResult`

The core sync generation method called by `invoke`, `generate`, and streaming operations. Returns a `ChatResult` wrapping `ChatGeneration` objects with `AIMessage` and optional metadata.

```python
def _generate(
    self,
    messages: list[BaseMessage],
    stop: list[str] | None = None,
    run_manager: CallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> ChatResult:
    """Generate chat response from messages."""
    # 1. Convert LangChain messages to provider format
    # 2. Build request payload from model params and kwargs
    # 3. Call provider API
    # 4. Parse provider response to AIMessage
    # 5. Extract and attach metadata (tokens, finish reason, etc.)
    # 6. Return ChatResult
```

For OpenAI, this method orchestrates:
1. Message normalization and conversion to OpenAI role/content format
2. Payload construction via `_get_request_payload()`
3. API invocation via the OpenAI SDK
4. Result parsing via `_create_chat_result()`

For Anthropic, the pattern is similar:
1. Format messages via `_format_messages()` to conform to Anthropic's schema
2. Build payload via `_get_request_payload()`
3. Call `_create()` on the synchronous client
4. Format output via `_format_output()`

#### `_llm_type` (property)

A string identifier for logging, serialization, and tracing (e.g., `"openai"`, `"anthropic"`, `"ollama"`). Must be unique within the ecosystem.

```python
@property
def _llm_type(self) -> str:
    """Return type of llm."""
    return "openai"
```

### Optional Streaming Support

Implementing these methods enables optimized streaming paths:

#### `_stream(messages, stop, run_manager, **kwargs) → Iterator[ChatGenerationChunk]`

If not implemented, `stream()` falls back to `invoke()`. Streaming is essential for real-time UX. OpenAI's `_stream` method:
- Sets `stream=True` on the request
- Iterates chunks from the provider's streaming API
- Converts each chunk to a `ChatGenerationChunk` with incremental content
- Fires callbacks (`run_manager.on_llm_new_token()`) for each chunk
- Handles provider errors and maps them to LangChain error classes

Anthropic's `_stream` similarly:
- Sets `stream=True` on the request
- Iterates event objects from the provider's streaming response
- Converts events to chunks via `_make_message_chunk_from_anthropic_event()`
- Accumulates usage metadata across chunks

#### `_agenerate(messages, stop, run_manager, **kwargs) → ChatResult`

Native async generation. If absent, `ainvoke` runs `_generate` in an executor. OpenAI and Anthropic both implement this to use their async clients for non-blocking I/O.

#### `_astream(messages, stop, run_manager, **kwargs) → AsyncIterator[ChatGenerationChunk]`

Async streaming. OpenAI wraps its async SDK's streaming with per-chunk timeouts (controlled via `stream_chunk_timeout`) to prevent indefinite hangs on stalled streams.

## Message Conversion

The boundary where provider APIs differ most: each partner must convert between LangChain's unified `BaseMessage` format and the provider's role/content schema.

### LangChain Message Types

Core message types in `langchain_core.messages`:
- **`HumanMessage`**: User input
- **`AIMessage`**: Model output; carries `tool_calls` (structured as `ToolCall` objects) and `additional_kwargs` (provider-specific fields like token counts, reasoning content)
- **`SystemMessage`**: Model instructions
- **`ToolMessage`**: Result from tool execution
- **`FunctionMessage`**: Legacy OpenAI function invocation result
- **Content**: String or list of content blocks (text, image, audio, file) for multimodal support

### Provider-Specific Conversion

OpenAI message conversion (in `_convert_dict_to_message` and related):
- `HumanMessage` → `{"role": "user", "content": ...}`
- `AIMessage` → `{"role": "assistant", "content": ..., "tool_calls": [...]}` (if tool calls present)
- `SystemMessage` → `{"role": "system" | "developer", "content": ...}` (respects `__openai_role__` field for developer role)
- `ToolMessage` → `{"role": "tool", "tool_call_id": ..., "content": ...}`
- `FunctionMessage` → `{"role": "function", "name": ..., "content": ...}` (legacy)

Anthropic message conversion (in `_format_messages` and helpers):
- Handles mid-conversation system messages (hoisting non-leading ones to the top-level `system` field)
- Normalizes tool call IDs for Anthropic compatibility (which rejects IDs with `.` and `:` characters)
- Merges runs of human/tool messages and deduplicates tool call blocks
- Supports special content blocks like `tool_addition` and `tool_removal` for mid-conversation tool changes
- Preserves cache control and citations metadata

Tool calls are parsed from the provider's response format (OpenAI uses `{"function": {"name": ..., "arguments": ...}}`, Anthropic uses `{"type": "tool_use", "name": ..., "input": ...}`) and wrapped as LangChain `ToolCall` objects carrying `id`, `name`, and `args`.

Reverse conversion (provider response → `AIMessage`) handles:
- Extracting text content and tool invocation data
- Parsing provider-specific metadata (token counts, finish reason, reasoning content, audio)
- Storing non-standard fields in `additional_kwargs` or content blocks for later retrieval

### Streaming Delta Conversion

Streaming requires incremental parsing. OpenAI's `_convert_delta_to_message_chunk` parses chunks containing partial text, tool call deltas (partial JSON), and token usage, accumulating them into `ChatGenerationChunk` objects that carry `tool_call_chunks` for efficient merging.

## Tool Calling Integration

`bind_tools(tools, tool_choice, **kwargs)` binds tool schemas to a model, returning a wrapped `Runnable` that encodes tools in the API request.

### Schema Binding

OpenAI's `bind_tools` accepts tools as:
- `BaseTool` instances (from `langchain_core.tools`)
- Pydantic models (with docstrings and field descriptions)
- Functions (analyzed for signature and docstring)
- Dicts (raw tool schemas)

Each is converted to the provider's format via `convert_to_openai_tool()` (from `langchain_core.utils.function_calling`), which produces JSON Schema representations compatible with the provider's API.

Anthropic's `bind_tools` similarly accepts the same formats and converts them using either `convert_to_openai_tool()` (for interop) or custom Anthropic-specific conversion.

### Function Calling Flow

1. **Binding**: User calls `model.bind_tools([GetWeather, GetPopulation])`
2. **Schema conversion**: Each tool is converted to the provider's format
3. **Request encoding**: Tools are passed to the provider API alongside the message
4. **Response parsing**: Provider returns tool calls; parsers extract them into `ToolCall` objects
5. **Tool execution**: Framework executes each tool; results become `ToolMessage` objects
6. **Feedback**: `ToolMessage` objects are added to the message history for next model call

### Tool Choice Control

`tool_choice` parameter steers the model:
- `"none"`: Never call a tool
- `"auto"`: Model decides
- `"any"` / `"required"` / `True`: Force at least one tool call
- `{"type": "function", "function": {"name": "GetWeather"}}` (OpenAI) or `{"type": "tool", "name": "GetWeather"}` (Anthropic): Force a specific tool
- `None` / `False`: Use provider default

OpenAI normalizes aliases (e.g., `"any"` → `"required"`) and handles both string shortcuts and dicts. Anthropic similarly normalizes and supports its own dict format.

### Strict Mode

When `strict=True`, the model is constrained to output exactly matching the JSON Schema. This is enforced server-side by compatible providers and requires schema validation. Enabling strict mode on OpenAI also requires all tools to be strict-compatible (relevant for Chat Completions API with response formats).

## Structured Output Support

`with_structured_output(schema, method="function_calling")` wraps a model to return outputs conforming to a schema, without requiring explicit tool definitions.

### Methods

OpenAI supports three methods:

#### `"function_calling"` (Default)

Treats the schema as a single tool and forces the model to call it. The `PydanticToolsParser` extracts the tool argument, yielding the parsed Pydantic instance (if schema is Pydantic) or a dict (if schema is JSON).

Advantages: Broad model support, simpler error handling.
Limitations: Requires schema compatibility with tool calling.

#### `"json_schema"`

Uses OpenAI's Structured Outputs API (JSON Schema mode), which enforces exact schema matching server-side. Requires `langchain-openai >= 0.3.0` and GPT models supporting the feature (not gpt-3.5, not gpt-4, yes to gpt-4o and newer).

Advantages: Strongest guarantee of schema compliance; supports complex schemas.
Limitations: Newer models only; higher API overhead.

#### `"json_mode"`

Requests JSON-formatted output via `response_format={"type": "json_object"}`. The model generates valid JSON but is **not** constrained to match the schema. Parsing is client-side via `JsonOutputParser`.

Advantages: Works with all chat models.
Limitations: Schema is a hint, not enforced; more parsing errors.

### Pydantic Binding

If schema is a Pydantic class:
- OpenAI generates a `with_structured_output` wrapper that applies `PydanticToolsParser` (for function_calling/json_schema) or `PydanticOutputParser` (for json_mode)
- Output is a Pydantic instance, fully validated
- Errors during parsing are raised or caught (if `include_raw=True`)

If schema is a dict:
- Output is a dict (no Pydantic validation)
- Parsing still uses JSON parsers but yields dicts

### Error Handling & Fallback

When `include_raw=True`, the wrapper returns a dict:
```python
{
    "raw": AIMessage(...),
    "parsed": <schema_instance> | None,
    "parsing_error": <exception> | None,
}
```

Parsing errors are caught and attached rather than raised, allowing graceful fallbacks.

## Configuration Patterns

Partners expose configuration across three layers:

### Constructor Parameters

Core model parameters are declared as Pydantic fields. OpenAI's example:

```python
class BaseChatOpenAI(BaseChatModel):
    model_name: str = Field(default="gpt-3.5-turbo", alias="model")
    temperature: float | None = None
    max_tokens: int | None = None
    top_p: float | None = None
    frequency_penalty: float | None = None
    presence_penalty: float | None = None
    openai_api_key: SecretStr | Callable[[], str] | None = Field(alias="api_key", default=None)
    openai_api_base: str | None = Field(alias="base_url", default=None)
    request_timeout: float | tuple[float, float] | None = Field(alias="timeout", default=None)
    max_retries: int | None = None
    model_kwargs: dict[str, Any] = Field(default_factory=dict)
```

Anthropic's example:

```python
class ChatAnthropic(BaseChatModel):
    model: str = Field(alias="model_name")
    max_tokens: int | None = Field(default=None, alias="max_tokens_to_sample")
    temperature: float | None = None
    anthropic_api_key: SecretStr = Field(default=SecretStr(""), alias="api_key")
    anthropic_api_url: str | None = Field(default=None, alias="base_url")
    default_request_timeout: float | None = Field(None, alias="timeout")
    max_retries: int = 2
    model_kwargs: dict[str, Any] = Field(default_factory=dict)
```

Key patterns:
- **`model_kwargs`**: Catch-all dict for unsupported parameters; passed directly to the provider API via `_build_model_kwargs()` which merges constructor kwargs with model_kwargs
- **Aliases**: Support alternative field names (e.g., `model_name` aliased to `"model"`, `api_key` aliased to `"openai_api_key"`)
- **Defaults**: Reasonable defaults inferred from environment or provider (e.g., `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`)
- **SecretStr**: Sensitive fields like API keys are automatically redacted in logs
- **Callables**: API keys can be sync or async callables for runtime refresh (`Callable[[], str]` or `Callable[[], Awaitable[str]]`)

### Environment Variable Fallback

Constructors read environment variables if explicit values are not provided. OpenAI's pattern:

```python
openai_api_key = Field(default=None, ...)  # Env: OPENAI_API_KEY
openai_api_base = Field(default=None, ...)  # Env: OPENAI_API_BASE, then OPENAI_BASE_URL
openai_organization = Field(default=None, ...) # Env: OPENAI_ORG_ID
```

Anthropic's pattern:

```python
anthropic_api_key = Field(default=SecretStr(""), alias="api_key")  # Env: ANTHROPIC_API_KEY
anthropic_api_url = Field(default=None, alias="base_url")  # Env: ANTHROPIC_API_URL, then ANTHROPIC_BASE_URL
```

The `from_env()` utility and Pydantic validators orchestrate this fallback. The `validate_environment()` method runs post-init to resolve credentials, validate configuration, and apply LangSmith gateway settings.

### Endpoint Overrides

Partners often support custom endpoints for proxies, fine-tuned models, or on-prem deployments:

```python
model = ChatOpenAI(
    base_url="http://localhost:8000/v1",  # vLLM, LM Studio, etc.
    api_key="fake-key"
)
```

Partners validate that overridden endpoints match expected schemas (e.g., OpenAI format compliance checks).

### Client Configuration

Advanced configuration for HTTP clients, proxies, and timeouts:

```python
model = ChatOpenAI(
    timeout=30.0,
    max_retries=2,
    http_client=my_httpx_client,  # Custom httpx.Client
    openai_proxy="http://proxy:8080",
)
```

OpenAI's implementation handles proxy auto-detection, socket options, and SSL verification, with fallback to sane defaults. Anthropic similarly supports custom HTTP clients and proxy configuration.

### Model-Specific Validation

Partners implement validators to detect model capabilities and enforce constraints. OpenAI's `validate_temperature` validator:
- For `o1` models: Sets `temperature=1` if not provided (o1 requires temperature=1)
- For `gpt-5` models (non-chat): Removes non-1 temperature values when reasoning is enabled

Anthropic enforces `thinking` parameter compatibility and validates `output_config.effort` values.

## Error Handling & Mapping

Partner packages map provider-specific errors to LangChain's standardized error hierarchy (from `langchain_core.exceptions`), enabling generic error catching while preserving provider context.

### OpenAI Error Mapping

```python
openai.BadRequestError → OpenAIInvalidRequestError(ModelInvalidRequestError)
openai.AuthenticationError → OpenAIAuthenticationError(ModelAuthenticationError)
openai.RateLimitError → OpenAIRateLimitError(ModelRateLimitError)
openai.APIConnectionError → OpenAIConnectionError(ModelConnectionError)
openai.APITimeoutError → OpenAITimeoutError(ModelTimeoutError)
openai.NotFoundError → OpenAIModelNotFoundError(ModelNotFoundError)
```

Special cases:
- Context window exceeded errors (detected by message content) → `OpenAIContextOverflowError(ContextOverflowError)`
- Structured output schema errors → Helpful warnings suggesting `method="function_calling"` fallback

### Anthropic Error Mapping

```python
anthropic.BadRequestError → AnthropicInvalidRequestError(ModelInvalidRequestError)
anthropic.AuthenticationError → AnthropicAuthenticationError(ModelAuthenticationError)
anthropic.RateLimitError → AnthropicRateLimitError(ModelRateLimitError)
anthropic.APIConnectionError → AnthropicConnectionError(ModelConnectionError)
anthropic.APITimeoutError → AnthropicTimeoutError(ModelTimeoutError)
anthropic.NotFoundError → AnthropicModelNotFoundError(ModelNotFoundError)
anthropic.OverloadedError → AnthropicOverloadedError(ModelAPIError)
```

Error handlers (`_handle_anthropic_bad_request`, `_handle_anthropic_api_error`) inspect error messages for specific conditions (e.g., "prompt is too long") and map to appropriate LangChain exceptions.

## Model Profiles

Partners can declare model capabilities (context window, supported modalities, tool calling support) in a **model profile registry**. OpenAI stores profiles in `data/_profiles.py`:

```python
_PROFILES = {
    "gpt-4o": {
        "max_input_tokens": 128000,
        "max_output_tokens": 4096,
        "supported_modalities": {"text", "image", "audio"},
        "tool_calling": True,
        "structured_output": True,
    },
    # ...
}
```

Anthropic stores profiles similarly in `data/_profiles.py`, declaring capabilities like `tool_calling`, `vision`, `output_config_support`, etc.

Profiles are loaded at model init via `_resolve_model_profile()` and exposed on the instance, enabling runtime capability checks:

```python
if model.profile and model.profile.get("tool_calling"):
    # Use bind_tools
```

## Async Support

Partner implementations should provide native async variants where possible:

- **`_agenerate()`**: Async generation using the provider's async client
- **`_astream()`**: Async streaming with proper timeout handling
- **Callable API keys**: Support `Callable[[], Awaitable[str]]` for async key fetching

OpenAI's async implementation uses `openai.AsyncOpenAI` and includes per-chunk streaming timeouts (`stream_chunk_timeout`) to prevent indefinite hangs on stalled streams. The timeout is controlled via environment variable `LANGCHAIN_OPENAI_STREAM_CHUNK_TIMEOUT_S` (default 120 seconds) or constructor parameter.

Anthropic similarly provides `_agenerate()` and `_astream()` using its async client, and `_aparse()` helper for parsing async responses.

## Version & Compatibility Management

Partners manage compatibility across:
1. **langchain-core versions**: Ensure dependency bounds match breaking changes
2. **Provider SDK versions**: Pin to avoid upstream API changes
3. **Python versions**: Declare supported Python ranges (e.g., `>=3.11.0,<4.0.0`)
4. **Model evolution**: Detect model capabilities and adapt behavior (e.g., strict mode support, Responses API availability, Anthropic's thinking support)

OpenAI uses validators to detect model capabilities (e.g., `validate_temperature` for o1/gpt-5 models) and warnings to guide users toward compatible alternatives.

Anthropic similarly adapts behavior based on model name (e.g., `claude-opus-5` vs `claude-opus-5-5`) for features like thinking, vision, and extended thinking.

## Extension Boundary

Partners are the extension point for:
- **Provider-specific features**: Built-in tools (web search, code execution), reasoning modes (thinking, extended thinking), audio I/O
- **Vendor variants**: Azure OpenAI, DeepSeek, OpenRouter (inheriting from base OpenAI or building independently)
- **Custom message formats**: Non-standard blocks (Anthropic thinking, reasoning content) stored in `additional_kwargs` or content lists
- **Mid-conversation tool changes**: Anthropic's `tool_addition`/`tool_removal` blocks enabling dynamic tool set updates

The core framework abstractions (BaseChatModel, BaseMessage, Runnable) remain stable; partners handle diversity at the integration boundary.

---

## Testing Patterns

### StandardTests Conformance

Partner packages use `langchain-tests` (in `libs/standard-tests/`) to run a standard test suite verifying:
- **Chat model contract**: `invoke`, `stream`, `batch`, async variants all work
- **Tool calling**: `bind_tools` produces correct schemas and parses responses
- **Structured output**: `with_structured_output` works with Pydantic/dict schemas
- **Serialization**: Models can be dumped/loaded via `to_dict()`/`from_dict()`
- **Callbacks**: Streaming callbacks fire at the right times

Example test registration (from OpenAI's conftest):

```python
def test_standard_base_model_params():
    """Test standard parameters work."""
    model = ChatOpenAI(model="gpt-4o-mini")
    assert model.invoke("hello") is not None
```

### Real vs. Mock API Testing

- **Real API tests**: Hit the actual provider (slow, require API key, counted against quota). Used for integration tests.
- **Mock/recorded tests**: Use `vcrpy` to record HTTP interactions and replay them. Fast, deterministic, no API key required.

OpenAI uses recorded cassettes in `tests/cassettes/` containing fixture responses for common scenarios.

### Test Categories

1. **Unit tests**: Isolated functionality (message conversion, schema validation, error mapping)
2. **Integration tests**: Full workflows with mocked API responses
3. **Snapshot tests** (via `syrupy`): Capture and diff model output, catching regressions in response parsing

## Standard Tests Usage

Partner packages integrate with `langchain-tests` to validate conformance to the core contract. Two test classes are typically implemented:

### Unit Tests

Created in `tests/unit_tests/test_standard.py`:

```python
from typing import Type
import pytest
from langchain_core.language_models import BaseChatModel
from langchain_tests.unit_tests import ChatModelUnitTests
from langchain_openai import ChatOpenAI

class TestChatOpenAIStandard(ChatModelUnitTests):
    @pytest.fixture
    def chat_model_class(self) -> Type[BaseChatModel]:
        return ChatOpenAI
```

These tests verify:
- Model instantiation with standard parameters
- Serialization/deserialization (`to_dict()`/`from_dict()`)
- Tool calling and structured output method signatures (without requiring live API calls)

### Integration Tests

Created in `tests/integration_tests/test_standard.py`:

```python
from typing import Type
import pytest
from langchain_core.language_models import BaseChatModel
from langchain_tests.integration_tests import ChatModelIntegrationTests
from langchain_openai import ChatOpenAI

class TestChatOpenAIStandard(ChatModelIntegrationTests):
    @pytest.fixture
    def chat_model_class(self) -> Type[BaseChatModel]:
        return ChatOpenAI
    
    @pytest.fixture
    def chat_model_params(self) -> dict[str, Any]:
        return {"model": "gpt-4o-mini"}
```

These test full workflows:
- `invoke`, `stream`, `batch`, and async variants (with real or mocked API)
- Tool calling end-to-end
- Structured output parsing and validation
- Callback firing during streaming
- Error handling with expected exception types

### Test Fixtures

- **`chat_model_class`** (required): The ChatModel subclass to test
- **`chat_model_params`**: Kwargs for constructor (e.g., model name, optional API key override)
- **`chat_model_has_tool_calling`** (auto-detected): Whether the model supports `bind_tools()`
- **`chat_model_has_structured_output`** (auto-detected): Whether the model supports `with_structured_output()`

## Handling Async API Keys

Partners supporting callable API keys enable runtime refresh and secrets management:

```python
# Sync callable
model = ChatOpenAI(
    model="gpt-4o-mini",
    api_key=lambda: os.environ.get("OPENAI_API_KEY")
)

# Async callable
async def get_api_key() -> str:
    # Fetch from secrets service, database, etc.
    return await secrets_client.get("openai_key")

model = ChatOpenAI(
    model="gpt-4o-mini",
    api_key=get_api_key
)
```

OpenAI's implementation extracts the key at invocation time, supporting both sync and async contexts.

## See Also

- [Architecture](/openwiki/architecture.md): Three-layer design where partners implement core abstractions
- [Chat Models](/openwiki/concepts/chat-models.md): BaseChatModel contract and streaming semantics
- [Messages](/openwiki/concepts/messages.md): Message types, content blocks, and conversion
- [OpenAI Provider](/openwiki/integrations/openai-provider.md): Detailed OpenAI integration walkthrough
