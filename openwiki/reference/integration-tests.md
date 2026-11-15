---
type: "Reference"
title: "In libs/partners/openai/tests/conftest.py"
openwiki_generated: true
verified:
  - by: openwiki/0.5.0
    at: 2026-11-15T08:15:26.682Z
sources:
  - id: openwiki-source-9069a5dd5fbb579fbd5470ce
    resource: repo://.github/workflows/integration_tests.yml
  - id: openwiki-source-711811fb8df6b32329091d23
    resource: repo://libs/partners/openai/tests/integration_tests/chat_models/test_base_standard.py
  - id: openwiki-source-bd29e79613d5f366a00068f5
    resource: repo://libs/standard-tests/langchain_tests/base.py
  - id: openwiki-source-db02c1dda8563ab005cd9d62
    resource: repo://libs/standard-tests/langchain_tests/conftest.py
  - id: openwiki-source-13cd70e2dd4252d8b05ac9c1
    resource: repo://libs/standard-tests/langchain_tests/integration_tests/__init__.py
  - id: openwiki-source-faf87f53de8944884a9cbd7c
    resource: repo://libs/standard-tests/langchain_tests/integration_tests/base_store.py
  - id: openwiki-source-5c8479432f53b0bc63aae72b
    resource: repo://libs/standard-tests/langchain_tests/integration_tests/cache.py
  - id: openwiki-source-f7c26a6473517c7b426ef023
    resource: repo://libs/standard-tests/langchain_tests/integration_tests/chat_models.py
  - id: openwiki-source-fd6affee984d6ca1f8e27e7d
    resource: repo://libs/standard-tests/langchain_tests/integration_tests/embeddings.py
  - id: openwiki-source-1f96085836b8ff843a9cff83
    resource: repo://libs/standard-tests/langchain_tests/integration_tests/retrievers.py
  - id: openwiki-source-281ce2ed3977e4fb9ce01038
    resource: repo://libs/standard-tests/langchain_tests/integration_tests/sandboxes.py
  - id: openwiki-source-88ae7b8133517e0f91f79329
    resource: repo://libs/standard-tests/langchain_tests/integration_tests/tools.py
  - id: openwiki-source-ae54d28347f4a08365be4997
    resource: repo://libs/standard-tests/langchain_tests/integration_tests/vectorstores.py
generated: { by: "openwiki/0.5.0", at: "2026-11-15T08:15:26.682Z" }
---


## Overview

The `langchain-tests` package provides a comprehensive suite of integration test utilities for building and validating LangChain implementations. It includes:

- **Standard test base classes** for chat models, embeddings, vector stores, retrievers, tools, and caches
- **VCR fixture configuration** for recording and replaying HTTP interactions
- **Sandbox integration tests** for deepagents-based backend implementations
- **Test classification** via pytest markers for selective execution

This reference documents the integration testing APIs available in `/libs/standard-tests/langchain_tests/integration_tests/`.

## Standard Test Base Classes

### Overview and Inheritance

All standard test classes inherit from `BaseStandardTests`, which enforces a contract: implementations must not override or delete standard tests without explicitly marking them with `@pytest.mark.xfail(reason="...")`. This ensures test suites remain complete and changes are intentional.

```python
class BaseStandardTests:
    """Base class for all standard tests."""
    
    def test_no_overrides_DO_NOT_OVERRIDE(self) -> None:
        """Validates that no standard tests are overridden without xfail marks."""
```

### ChatModelIntegrationTests

Tests chat model implementations against real APIs. Inherits from `ChatModelTests` (unit test base).

**Required Properties:**

```python
class TestMyChatModel(ChatModelIntegrationTests):
    @property
    def chat_model_class(self) -> type[BaseChatModel]:
        """The chat model class to test, e.g., ChatOpenAI."""
        return MyChatModel
    
    @property
    def chat_model_params(self) -> dict:
        """Initialization parameters for the model."""
        return {"model": "gpt-4", "temperature": 0}
```

**Feature Control Properties:**

Optional properties control what features are tested:

```python
@property
def has_tool_calling(self) -> bool:
    """Whether the model supports tool calling. (Auto-detected if not overridden)"""
    return True

@property
def has_structured_output(self) -> bool:
    """Whether the model supports structured output."""
    return True

@property
def supports_image_inputs(self) -> bool:
    """Whether the model accepts image inputs in messages."""
    return True

@property
def enable_vcr_tests(self) -> bool:
    """Whether to enable VCR-backed cassette tests (default: False)."""
    return True

@property
def structured_output_kwargs(self) -> dict:
    """Additional kwargs for with_structured_output() calls."""
    return {}
```

**Test Coverage:**

Provides 70+ standard tests covering:
- Basic invocation and streaming
- Tool calling and tool choice
- Structured output generation
- Multi-modality (images, audio, video)
- Message types and content blocks
- Batch operations

### EmbeddingsIntegrationTests

Tests embeddings model implementations.

**Required Properties:**

```python
class TestMyEmbeddings(EmbeddingsIntegrationTests):
    @property
    def embeddings_class(self) -> type[Embeddings]:
        return MyEmbeddingsModel
    
    @property
    def embedding_model_params(self) -> dict:
        return {"model": "embedding-001"}
```

**Standard Tests:**

- `test_embed_query()` — Embed single string queries
- `test_embed_documents()` — Embed document batches
- `test_aembed_query()` — Async query embedding
- `test_aembed_documents()` — Async document embedding

### VectorStoreIntegrationTests

Tests vector store implementations with document add/search operations.

**Required Fixture:**

```python
class TestMyVectorStore(VectorStoreIntegrationTests):
    @pytest.fixture
    def vectorstore(self) -> VectorStore:
        """Return an empty VectorStore. Fixture is reset after each test."""
        store = MyVectorStore(embedding_function=self.get_embeddings())
        try:
            yield store
        finally:
            store.delete_collection()
```

**Feature Control:**

```python
@property
def has_sync(self) -> bool:
    """Enable/disable sync tests (default: True)."""
    return True

@property
def has_async(self) -> bool:
    """Enable/disable async tests (default: True)."""
    return True

@property
def has_get_by_ids(self) -> bool:
    """Whether the store supports get_by_ids() (default: True)."""
    return True

@staticmethod
def get_embeddings() -> Embeddings:
    """Return embeddings model. (Provided: DeterministicFakeEmbedding)."""
```

### RetrieversIntegrationTests

Tests retriever implementations.

**Required Properties:**

```python
class TestMyRetriever(RetrieversIntegrationTests):
    @property
    def retriever_constructor(self) -> type[BaseRetriever]:
        return MyRetriever
    
    @property
    def retriever_constructor_params(self) -> dict[str, Any]:
        return {"index": my_index, "top_k": 5}
    
    @property
    def retriever_query_example(self) -> str:
        """Example query string for testing."""
        return "What is machine learning?"
    
    @property
    def num_results_arg_name(self) -> str:
        """Parameter name for result count (default: 'k')."""
        return "k"
```

### ToolsIntegrationTests

Tests tool implementations.

**Required Properties:**

```python
class TestMyTool(ToolsIntegrationTests):
    @property
    def tool_invoke_params_example(self) -> dict:
        """Example parameters for tool invocation."""
        return {"query": "example"}
```

**Standard Tests:**

- `test_invoke_matches_output_schema()` — Tool returns valid ToolMessage
- `test_invoke_no_tool_call()` — Tool handles dict kwargs without error
- `test_async_invoke_matches_output_schema()` — Async invoke returns valid message

### BaseStoreSyncTests / BaseStoreAsyncTests

Tests key-value store implementations (`BaseStore` abstraction).

**Required Fixtures:**

```python
class TestMyStore(BaseStoreSyncTests[str]):
    @pytest.fixture
    def kv_store(self) -> BaseStore[str, MyValue]:
        """Return an empty key-value store."""
        store = MyStore()
        try:
            yield store
        finally:
            store.clear()
    
    @pytest.fixture
    def three_values(self) -> tuple[MyValue, MyValue, MyValue]:
        """Return three example values for testing."""
        return ("value1", "value2", "value3")
```

### SyncCacheTestSuite / AsyncCacheTestSuite

Tests cache implementations (`BaseCache` abstraction).

**Required Fixture:**

```python
class TestMyCache(SyncCacheTestSuite):
    @pytest.fixture
    def cache(self) -> BaseCache:
        """Return an empty cache."""
        c = MyCache()
        try:
            yield c
        finally:
            c.clear()
```

**Standard Tests:**

- `test_cache_is_empty()` — Cache starts empty
- `test_update_cache()` — Cache stores and retrieves generations
- `test_cache_hit()` — Lookup finds stored values
- `test_cache_miss()` — Lookup returns None for missing keys
- `test_clear_cache()` — Clear removes all entries

## VCR Fixture Configuration

### Overview

VCR (Video Cassette Recorder) records and replays HTTP interactions for integration tests. Configuration is managed through pytest fixtures in `langchain_tests.conftest`.

### Base Configuration

The `base_vcr_config()` function returns the default VCR configuration:

```python
def base_vcr_config() -> dict[str, Any]:
    """Return VCR configuration for all cassettes."""
    return {
        "record_mode": "once",  # Record if missing, replay if exists
        "filter_headers": [
            ("authorization", "PLACEHOLDER"),
            ("x-api-key", "PLACEHOLDER"),
            ("api-key", "PLACEHOLDER"),
        ],
        "match_on": ["method", "uri", "body"],  # Request matching strategy
        "allow_playback_repeats": True,  # Allow replaying same cassette multiple times
        "decode_compressed_response": True,  # Auto-decompress gzip responses
        "cassette_library_dir": "tests/cassettes",  # Cassette storage location
        "path_transformer": VCR.ensure_suffix(".yaml"),
    }
```

**Record Modes:**

- `"once"` — Record on first run, replay subsequently (default)
- `"new_episodes"` — Record new interactions, replay existing
- `"none"` — Playback-only, fail if cassette missing
- `"all"` — Always record, overwriting previous

### Custom Serialization

`CustomSerializer` handles YAML and gzip compression for cassettes:

```python
class CustomSerializer:
    @staticmethod
    def serialize(cassette_dict: dict[str, Any]) -> bytes:
        """Convert cassette to gzip-compressed YAML."""
        # - Coerces io.BytesIO to raw bytes
        # - Uses yaml.safe_dump() for security
        # - Compresses result via gzip
        
    @staticmethod
    def deserialize(data: bytes) -> dict[str, Any]:
        """Decompress and parse YAML."""
        # - Decompresses gzip data
        # - Uses yaml.safe_load() (secure parsing)
        # - Reconstructs VCR Request objects
```

**File Format:**

- **Storage**: `tests/cassettes/` relative to test module
- **Naming**: Test module and function path, e.g., `test_chat_models/test_invoke.yaml.gz`
- **Compression**: gzip-compressed YAML for human inspection after decompression

### Sensitive Header Filtering

The `CustomSerializer` applies global header filtering before cassettes are saved:

**Base Filters (always applied):**

```
authorization → PLACEHOLDER
x-api-key → PLACEHOLDER
api-key → PLACEHOLDER
```

**Partner-Specific Filters:**

Libraries extend `base_vcr_config()` for provider-specific redactions:

```python
# In libs/partners/openai/tests/conftest.py
def base_vcr_config() -> dict[str, Any]:
    config = base_vcr_config_from_langchain_tests()
    config["filter_headers"].extend([
        # OAuth and JWT tokens
        ("authorization", "**REDACTED**"),
        ("refresh_token", "**REDACTED-JWT**"),
        # Recursive filtering of JSON body fields
    ])
    return config
```

### Cassette Fixture Usage

```python
@pytest.fixture
def vcr_config() -> dict[str, Any]:
    """Pytest fixture providing VCR configuration."""
    return base_vcr_config()

@pytest.mark.vcr
def test_chat_invoke(cassette: Cassette) -> None:
    """Test marked with @pytest.mark.vcr uses cassette playback."""
    # First run: records HTTP interactions
    # Subsequent runs: replays from cassette
    response = my_model.invoke("Hello")
    assert response is not None
```

**Cassette Lifecycle:**

1. Test decorated with `@pytest.mark.vcr` starts
2. If cassette exists: VCR loads and enters playback mode
3. HTTP requests are matched against cassette using `["method", "uri", "body"]`
4. Matching response is returned from cassette
5. If no cassette: VCR enters recording mode (if `record_mode != "none"`)
6. Responses are filtered, serialized, and saved

## Sandbox Integration Tests

### SandboxIntegrationTests

Tests implementations of the `deepagents.backends.protocol.SandboxBackendProtocol` — an abstraction for isolated code execution sandboxes.

**Availability:**

Requires `deepagents` package (`pytest.importorskip("deepagents")`):

```python
from langchain_tests.integration_tests import SandboxIntegrationTests
```

**Required Abstract Fixture:**

```python
class TestMySandboxBackend(SandboxIntegrationTests):
    @pytest.fixture(scope="class")
    def sandbox(self) -> Iterator[SandboxBackendProtocol]:
        """Yield a clean sandbox backend and tear it down."""
        backend = create_my_sandbox()
        try:
            yield backend
        finally:
            backend.delete()
```

### Sandbox Backend Protocol

The `SandboxBackendProtocol` (from `deepagents.backends.protocol`) defines standard methods for sandbox file operations and command execution.

**File Operations:**

```python
# Write a new file (fails if already exists)
result: WriteResult = sandbox_backend.write(path, content)

# Read file contents
result: ReadResult = sandbox_backend.read(path)
# Returns base64-encoded content for binary files

# Edit file (replace text)
result: EditResult = sandbox_backend.edit(path, old_text, new_text)

# List directory contents
result: ListResult = sandbox_backend.ls(directory)

# Find files by glob pattern
result: GlobResult = sandbox_backend.glob(pattern, path=directory)

# Search file contents (literal matching)
result: GrepResult = sandbox_backend.grep(text, path=directory)
```

**Upload/Download:**

```python
# Upload files as bytes
responses: list[FileUploadResponse] = sandbox_backend.upload_files(
    [(path1, bytes1), (path2, bytes2)]
)

# Download files as bytes
responses: list[FileDownloadResponse] = sandbox_backend.download_files(
    [path1, path2]
)
```

**Command Execution:**

```python
result: ExecuteResponse = sandbox_backend.execute(command_string)
# result.exit_code — Process exit code
# result.output — stdout + stderr combined
# result.truncated — Whether output exceeded budget
```

### Test Coverage

**File Operations:**

- `test_write_new_file()` — Write and read file via command
- `test_read_basic_file()` — Read file content
- `test_read_binary_file()` — Base64-encoded binary reads
- `test_read_binary_file_100_kib()` — Large binary file handling
- `test_read_binary_file_1_mib_returns_error()` — Size limit enforcement (512 KiB)
- `test_edit_single_occurrence()` — Replace text in file
- `test_write_creates_parent_dirs()` — Nested directory creation
- `test_write_existing_file_fails()` — Prevent overwrite
- `test_write_special_characters()` — Preserve shell metacharacters
- `test_write_empty_file()` — Handle empty content
- `test_write_unicode_content()` — Unicode preservation
- `test_write_very_long_content()` — Large content handling

**Directory Operations:**

- `test_ls_lists_files()` — List directory contents
- `test_glob()` — Pattern matching on file paths
- `test_grep_literal()` — Literal text search in files

**Upload/Download:**

- `test_upload_single_file()` — Single file upload
- `test_download_single_file()` — Single file download
- `test_upload_download_roundtrip()` — Bytes integrity
- `test_upload_multiple_files_order_preserved()` — Multi-file ordering
- `test_download_multiple_files_order_preserved()` — Multi-file ordering
- `test_upload_binary_content_roundtrip()` — Binary integrity
- `test_upload_large_file_reports_expected_size()` — 10 MiB file handling
- `test_download_error_file_not_found()` — Missing file error
- `test_download_error_is_directory()` — Directory download error
- `test_download_error_permission_denied()` — Permission error
- `test_download_error_invalid_path_relative()` — Relative path rejection
- `test_upload_relative_path_returns_invalid_path()` — Relative path rejection

**Command Execution:**

- `test_execute_large_stdout_payload()` — 500 KiB output handling
- `test_execute_capture_at_source_offload()` — Large result offloading via CompositeBackend

### Configuration Properties

```python
class SandboxIntegrationTests(BaseStandardTests):
    @property
    def sandbox_root_dir(self) -> str:
        """Base directory for file operations (default: /tmp/test_sandbox_ops/)."""
        return "/tmp/test_sandbox_ops/"
    
    def sandbox_path(self, relative_path: str, *, root_dir: str | None = None) -> str:
        """Build paths under the sandbox root."""
        ...
    
    @property
    def has_sync(self) -> bool:
        """Enable/disable sync tests (default: True)."""
        return True
    
    @property
    def has_async(self) -> bool:
        """Enable/disable async tests (default: True)."""
        return True
```

### Fixture Lifecycle

```python
@pytest.fixture(scope="class")
def sandbox_backend(
    self, sandbox: SandboxBackendProtocol
) -> SandboxBackendProtocol:
    """Provides the configured sandbox backend."""
    return sandbox

@pytest.fixture(autouse=True)
def sandbox_test_root(self, request: pytest.FixtureRequest) -> str:
    """Creates isolated test directory for each test case."""
    # Returns unique path like: /tmp/test_sandbox_ops/test_write_new_file
    # Automatically injected into all test methods
```

### Advanced: CompositeBackend and Output Offloading

For large command output, `deepagents.backends.CompositeBackend` handles offloading to files:

```python
from deepagents.backends import CompositeBackend

backend = CompositeBackend(
    default=sandbox_backend,
    routes={},
    artifacts_root=sandbox_test_root  # Where to store large outputs
)

# Command producing ~225 KiB output is offloaded:
# - Tool message contains head/tail preview + read_file pointer
# - Full output saved to artifacts_root/large_tool_results/<tool-id>
```

## Integration Test Execution

### Pytest Markers

Tests are classified using markers for selective execution:

```python
@pytest.mark.vcr
def test_with_cassette():
    """Runs with VCR cassette playback."""

@pytest.mark.scheduled
def test_live_api():
    """Runs on daily schedule against live API."""

def test_offline():
    """Runs as unit-like test without network."""
```

### Running Tests Locally

```bash
# Full integration test suite (live + cassette)
cd libs/partners/openai
make integration_tests

# VCR-backed tests only (no API keys)
make test_vcr

# Unit-like offline tests
make test

# Specific test with live API
pytest tests/integration_tests/chat_models/test_base.py::test_invoke -m scheduled
```

### Recording New Cassettes

```bash
# Record new interactions in cassettes/
pytest tests/integration_tests/ --record-mode=new_episodes

# Cassettes are committed to repo; team uses playback-only mode
```

## Related Resources

- **Integration Testing Guide**: `/openwiki/testing/integration-tests.md` — Best practices and CI/CD integration
- **Unit Testing Guide**: `/openwiki/testing/unit-tests.md` — Unit test utilities
- **langchain-tests Package**: `repo://libs/standard-tests/` — Source code
