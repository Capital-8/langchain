---
type: "Developer Tools & Commands"
title: "Development Commands and Local Setup"
description: "Quick reference for uv, make, lint, test, and type-checking commands in the LangChain monorepo, including environment setup, pre-commit hooks, and testing workflows."
tags: [development, build, testing, linting, typing, uv, make, pre-commit, local-setup]
verified:
  - by: openwiki/0.5.0
    at: 2026-11-15T08:15:26.682Z
sources:
  - id: openwiki-source-4d1645cb6317345817452838
    resource: repo://.pre-commit-config.yaml
  - id: openwiki-source-8f1875229ad4a704c8e20a06
    resource: repo://libs/core/Makefile
  - id: openwiki-source-3486a94e6eb23a78271a5bfb
    resource: repo://libs/core/pyproject.toml
  - id: openwiki-source-f708a9db48bfcf1b154e4708
    resource: repo://libs/langchain/Makefile
  - id: openwiki-source-f4436232e0451a04247e92e5
    resource: repo://libs/langchain/pyproject.toml
  - id: openwiki-source-49fbcc45434b619b68220bf9
    resource: repo://libs/Makefile
  - id: openwiki-source-a6e669bb11f217c6fbd06670
    resource: repo://libs/partners/anthropic/Makefile
generated: { by: "openwiki/0.5.0", at: "2026-11-15T08:15:26.682Z" }
---

## Overview

The LangChain Python monorepo uses `uv` for dependency management, `make` for task automation, and `ruff`/`mypy` for code quality. This page provides a quick reference for common development commands and setup workflows.

## Initial Setup

### Install Dependencies Per Package

Each package in `libs/` has its own `pyproject.toml` and `uv.lock`. Before running tests or making changes, set up dependencies:

```bash
# Navigate to package directory
cd libs/core

# Install all dependency groups (lint, typing, test, dev)
uv sync --all-groups

# Or install only specific groups
uv sync --group test
uv sync --group lint
uv sync --group typing
```

The `--all-groups` flag ensures you have tools for linting, type checking, and testing. Common dependency groups defined in `pyproject.toml`:

- **lint**: Ruff linter and formatter
- **typing**: Mypy type checker and stubs
- **test**: Pytest and testing utilities
- **test_integration**: Integration test dependencies (API clients, mock servers)
- **dev**: Development tools (Jupyter, setuptools)

### uv Essentials

`uv` is a fast Python package installer and dependency resolver that replaces `pip`, `poetry`, and `conda`. Key principles:

- **Automatic environment management**: `uv sync` and `uv run` manage virtual environments automatically—do not manually source `.venv/bin/activate`
- **Per-package isolation**: Each package in `libs/` maintains its own environment via `uv.lock`; do not create ad-hoc virtual environments
- **Explicit dependency groups**: Use group selectors for faster setup
- **Frozen lock files**: `UV_FROZEN=true` in Makefiles prevents unintended changes

Common uv commands:

```bash
# Sync dependencies (respects groups and lock file)
uv sync --all-groups

# Run commands in package environment
uv run --group test pytest tests/unit_tests/

# Add a dependency temporarily for testing
uv run --with 'anthropic>=1.0.0' --group test pytest tests/integration_tests/

# Update lock file (when dependencies change)
uv lock

# Check lock file is up-to-date
uv lock --check
```

### Pre-Commit Setup

The repository uses pre-commit hooks to enforce code quality automatically on commit. Install once per repository:

```bash
pre-commit install
```

Pre-commit then runs automatically on staged files before each commit. To manually trigger:

```bash
# Run all hooks on all files
pre-commit run --all-files

# Run a specific hook by ID
pre-commit run core --all-files
pre-commit run lint --all-files
```

#### Bypassing Pre-Commit Hooks

To skip hooks temporarily (use with caution):

```bash
# Skip all hooks for this commit
git commit --no-verify

# Skip specific hooks via SKIP environment variable
SKIP=core,langchain pre-commit run --all-files
```

## Pre-Commit Hooks Details

The `.pre-commit-config.yaml` defines comprehensive hooks that enforce:

### Standard Validation
- **YAML/TOML syntax**: `check-yaml`, `check-toml`
- **File formatting**: Ensures files end with newline, removes trailing whitespace
- **Text normalization**: Fixes curly quotes and non-standard spaces

### Per-Package Format and Lint
Each package in `libs/` has a local hook running:
```
make -C libs/<package> format lint
```

This applies ruff formatting, linting, and mypy type checking. Hook IDs in pre-commit:
- `core`: libs/core
- `langchain`: libs/langchain  
- `langchain_v1`: libs/langchain_v1
- `standard-tests`: libs/standard-tests
- `text-splitters`: libs/text-splitters
- Partner packages: `anthropic`, `chroma`, `exa`, `fireworks`, `groq`, `huggingface`, `mistralai`, `nomic`, `ollama`, `openai`, `qdrant`, etc.

### Version Consistency Checks
Each package with a version file has a hook ensuring `pyproject.toml` version matches source code:
```
make -C libs/<package> check_version
```

Examples: `core-version`, `langchain-v1-version`, `anthropic-version`, etc.

### Branch Protection
Prevents direct commits to `master` branch.

## Linting and Formatting

### Complete Lint Suite

```bash
# From package directory
make lint
```

This runs four checks in sequence:
1. **Import organization**: `./scripts/lint_imports.sh` (validates imports are properly structured)
2. **Ruff check**: Logical errors, naming conventions, code smells (via `ruff check`)
3. **Ruff format (diff mode)**: Format checking without modifications (`ruff format --diff`)
4. **Mypy**: Static type checking with strict mode enabled

### Format Code

```bash
make format
```

Applies auto-fixes:
- `ruff format`: Formats code (imports, spacing, etc.)
- `ruff check --fix`: Auto-fixes linting issues (unsorted imports, unused variables)

**Note**: Always run `make format` before `make lint` since formatting can resolve some lint issues.

### Ruff-Only Commands

For faster iteration during development, run ruff separately:

```bash
# Check without fixing
uv run --group lint ruff check .
uv run --group lint ruff format . --diff

# Apply fixes
uv run --group lint ruff check . --fix
uv run --group lint ruff format .
```

Ruff processes Python files and Jupyter notebooks. Run from within a package directory.

### Lint Changes Only

For faster iteration, lint only files changed in your branch:

```bash
make lint_diff
make format_diff
```

These targets run the linters and formatters only on files that differ from `master` in the current package directory. Useful for incremental changes.

### Lint Specific Paths

```bash
# Lint only package source code (not tests)
make lint_package

# Lint only test code
make lint_tests
```

## Type Checking

### Run mypy

```bash
make type
```

This runs mypy with strict mode enabled. Mypy is configured in each package's `pyproject.toml` with:
- Strict mode (`strict = true`)
- Pydantic plugin enabled
- Unreachable code warnings enabled

### Type Checking with Lint

Type checking is included as the fourth step in `make lint`, so running the full lint suite checks types automatically.

## Testing

### Run Unit Tests

Unit tests in `tests/unit_tests/` run with network isolation:

```bash
# From package directory
make test
```

This runs:
- `pytest` with xdist parallelization (`-n auto`)
- Network disabled (`--disable-socket --allow-unix-socket`) to prevent accidental external calls
- Socket restrictions disabled for Unix sockets only (localhost communication works)
- Benchmark disabled for speed

### Run Specific Test

```bash
# Using make variable
make test TEST_FILE=tests/unit_tests/test_specific.py

# Or direct pytest
uv run --group test pytest tests/unit_tests/test_specific.py
```

### Run Tests in Watch Mode

```bash
# Auto-rerun on file changes
make test_watch
```

Uses `pytest-watcher` to automatically rerun tests when source files change. Handy for TDD workflows.

### Extended Tests

```bash
# Run tests marked with @pytest.mark.requires_python_3_11_plus or similar
make extended_tests
```

Tests marked as "extended" (e.g., slow or resource-intensive tests) are skipped by default but included with this target.

### Integration Tests

Integration tests in `tests/integration_tests/` require additional dependencies and allow network access:

```bash
# Run integration tests
make integration_tests
```

This runs:
- `pytest` with both `test` and `test_integration` groups
- Network enabled (no socket restrictions)
- Allows testing against external APIs

**Testing with multiple SDK versions** (example: anthropic package):

```bash
# Test against multiple anthropic versions
uv run --with 'anthropic>=0.120.0,<1.0.0' --group test --group test_integration pytest tests/integration_tests/
uv run --with 'anthropic>=1.0.0,<2.0.0' --group test --group test_integration pytest tests/integration_tests/
```

### Coverage Reports

```bash
# Generate coverage report with text and XML output
make coverage
```

Produces:
- Terminal output showing covered/uncovered lines
- XML report for CI tools
- HTML report (in some configurations)

## Monorepo-Level Tasks

The root `libs/Makefile` provides commands that operate across multiple packages:

```bash
# Update lock files for all core packages
cd libs && make lock

# Verify all lock files are up-to-date
cd libs && make check-lock
```

These iterate over `core`, `text-splitters`, `langchain`, and `langchain_v1`, ensuring consistent lock states across the monorepo.

## Common Development Workflows

### Local Development Loop

```bash
# 1. Navigate to your package
cd libs/core

# 2. Install dependencies once
uv sync --all-groups

# 3. Make changes to code

# 4. Run tests in watch mode (auto-rerun on save)
make test_watch

# 5. When ready to commit, format and lint
make format
make lint

# 6. Run full test suite before push
make test
```

### Pre-commit Enforcement

Before pushing, pre-commit hooks run automatically:

```bash
# Simulate what git will check
pre-commit run --all-files

# If pre-commit fails, fix issues:
make format  # Fix formatting
make lint    # Check for remaining issues
git add .
git commit -m "Fix lint/format issues"
```

### Integration Testing with Multiple Versions

For partner packages that support multiple SDK versions:

```bash
cd libs/partners/anthropic

# Test smoke tests against multiple anthropic versions
uv run --with 'anthropic>=0.120.0,<1.0.0' --group test --group test_integration pytest tests/integration_tests/test_chat_models.py::test_stream

uv run --with 'anthropic>=1.0.0,<2.0.0' --group test --group test_integration pytest tests/integration_tests/test_chat_models.py::test_stream
```

### Faster Iteration with lint_diff and format_diff

When working on a feature branch:

```bash
# Format only changed files (vs master)
make format_diff

# Lint only changed files
make lint_diff

# This is much faster than linting the entire package
```

### Debugging Test Failures

```bash
# Run with verbose output
uv run --group test pytest tests/unit_tests/test_foo.py -vv

# Run a single test function
uv run --group test pytest tests/unit_tests/test_foo.py::test_something -vv

# Disable socket isolation temporarily for debugging (allows network)
uv run --group test pytest tests/unit_tests/test_foo.py --disable-socket=false -vv

# Run with pdb debugger on failure
uv run --group test pytest tests/unit_tests/test_foo.py --pdb
```

## Environment Variables and Configuration

### Test Environment Cleanup

The `test` target unsets several environment variables to prevent accidental external service calls:

```bash
env \
  -u LANGCHAIN_TRACING_V2 \
  -u LANGCHAIN_API_KEY \
  -u LANGSMITH_API_KEY \
  -u LANGSMITH_TRACING \
  -u LANGCHAIN_PROJECT \
  uv run --group test pytest ...
```

This ensures unit tests run in isolation without connecting to LangSmith or other external services.

### Lock File Behavior

`UV_FROZEN=true` is set via `.EXPORT_ALL_VARIABLES` in all Makefiles:

```makefile
.EXPORT_ALL_VARIABLES:
UV_FROZEN = true
```

This prevents `uv sync` from modifying lock files during CI or development. To update dependencies:

```bash
# Explicitly unlock and update
cd libs/core
uv lock  # Updates uv.lock

# Or from monorepo root
cd libs && make lock
```

## Troubleshooting

### "ModuleNotFoundError" or missing dependencies

```bash
# Ensure you've synced dependencies for your package
cd libs/core
uv sync --all-groups
```

### Pre-commit hook skips your changes

Make sure your files match the hook patterns. For example, `core` hook only runs on `^libs/core/` paths. If you're in `libs/langchain`, the core hook won't trigger.

### Type checking errors after code changes

Clear the mypy cache:

```bash
rm -rf .mypy_cache
make type
```

### Lock file is stale

```bash
cd libs && make check-lock
# If it fails, update:
cd libs && make lock
```

### Too many test dependencies

If you only need to run tests without linting:

```bash
uv sync --group test
```

This is faster for quick iteration; later sync `--all-groups` before committing.
