---
type: "Reference"
title: "CI Workflow Configuration Reference"
description: "Complete technical reference for GitHub Actions workflows, job definitions, matrix strategies, reusable workflow patterns, environment variables, and secret management in LangChain's multi-package CI/CD pipeline."
tags: [ci, workflows, github-actions, reference, automation, python-matrix, dependency-management]
verified:
  - by: openwiki/0.5.0
    at: 2026-11-15T08:15:26.682Z
sources:
  - id: openwiki-source-3638ff2c170b760df159d81b
    resource: repo://.github/actions/uv_setup/action.yml
  - id: openwiki-source-34e57b5a3a0c875639ab72a7
    resource: repo://.github/scripts/check_diff.py
  - id: openwiki-source-f35e7c44cc1805709393a581
    resource: repo://.github/workflows/_lint.yml
  - id: openwiki-source-c92cc62695c6def991956428
    resource: repo://.github/workflows/_release.yml
  - id: openwiki-source-c9c292f4ecabe180cdae27ce
    resource: repo://.github/workflows/_test_pydantic.yml
  - id: openwiki-source-d8a8900818f4abab719bd1b7
    resource: repo://.github/workflows/_test_vcr.yml
  - id: openwiki-source-4d9cccca7700db7220ec055e
    resource: repo://.github/workflows/_test.yml
  - id: openwiki-source-6e3a52c89729b5704dbd7eec
    resource: repo://.github/workflows/check_diffs.yml
  - id: openwiki-source-9069a5dd5fbb579fbd5470ce
    resource: repo://.github/workflows/integration_tests.yml
generated: { by: "openwiki/0.5.0", at: "2026-11-15T08:15:26.682Z" }
---

## Overview

LangChain's CI/CD infrastructure is built on three interconnected GitHub Actions workflows that enforce code quality, handle dependency compatibility testing, and manage safe package releases:

- **`check_diffs.yml`** — Primary PR CI with intelligent change detection and dynamic test matrix generation
- **`integration_tests.yml`** — Scheduled daily integration testing against live APIs with manual trigger support
- **`_release.yml`** — Multi-stage release pipeline with validation gates and PyPI publication

Supporting reusable workflows define job execution patterns for linting, unit testing, Pydantic compatibility, VCR playback testing, integration compilation checks, and extended test suites.

---

## Core CI Workflow: `check_diffs.yml`

### Trigger Configuration

Runs on:
- Every pull request (opened, synchronized, reopened)
- All pushes to `master` branch
- Merge queue participation (GitHub's protected merge feature)

```yaml
on:
  push:
    branches: [master]
  pull_request:
  merge_group:
```

### Concurrency & Resource Optimization

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Cancels redundant runs within the same branch/PR. If another push arrives while the workflow is running, the earlier run is cancelled to conserve runner capacity.

### Global Environment Variables

```yaml
env:
  UV_FROZEN: "true"      # Enforce frozen uv.lock (no dependency updates)
  UV_NO_SYNC: "true"     # Skip sync for faster setup in build job
```

- `UV_FROZEN=true` prevents automatic dependency resolution changes
- `UV_NO_SYNC=true` skips the expensive sync step in the build job, which only needs to analyze files, not install dependencies

### Permissions

```yaml
permissions:
  contents: read
```

Minimal read-only access; workflows cannot write to the repository.

---

## Job: `build` — Change Detection & Matrix Generation

**Purpose:** Analyze changed files and generate dynamic test matrices for downstream jobs.

**Triggers:** Skips if PR has `ci-ignore` label (`if: ${{ !contains(github.event.pull_request.labels.*.name, 'ci-ignore') }}`)

**Environment:** `ubuntu-latest`

### Execution Steps

1. **Checkout Code** — `actions/checkout@v6`
2. **Setup Python 3.11** — `actions/setup-python@v7.1.0`
3. **Get Changed Files** — `Ana06/get-changed-files@v2.3.0` with JSON format output
4. **Analyze Changes & Generate Matrix** — Python script `check_diff.py`
   - Installs: `packaging`, `requests`
   - Runs unit tests: `python -m unittest discover -s .github/scripts -p test_check_diff.py`
   - Generates matrix: `python .github/scripts/check_diff.py "$ALL_CHANGED_FILES" >> $GITHUB_OUTPUT`

### Output Matrices (JSON arrays of job configurations)

Each output is a JSON array suitable for GitHub Actions matrix expansion:

- **`lint`** — Linting configurations: `[{working-directory, python-version}, ...]`
- **`test`** — Unit test configurations
- **`test-pydantic`** — Pydantic version matrix with min/max versions per package
- **`extended-tests`** — Extended test suite configurations
- **`compile-integration-tests`** — Integration test compilation checks (import validation only)
- **`vcr-tests`** — VCR cassette-backed integration tests (playback mode)
- **`dependencies`** — Dependency compatibility checks

### Change Detection Logic

File analysis rules (from `check_diff.py`):

**Core packages identified:**
- `libs/core`
- `libs/text-splitters`
- `libs/langchain`
- `libs/langchain_v1`
- `libs/model-profiles`

**Detection flow:**

1. Maps file changes → affected package directories
2. Builds dependency graph from `pyproject.toml` and `extended_testing_deps.txt`
3. Cascades changes through dependent packages (e.g., core change triggers all dependents)
4. Ignores partners from dependents: `huggingface` (CI instability) — still runs if directly edited
5. **Infrastructure changes trigger extended tests on all core packages:**
   - Changes to `.github/workflows/`, `.github/actions/`, `.github/scripts/check_diff.py` automatically add all `LANGCHAIN_DIRS` to extended tests
   - Conservative default: ensures CI infrastructure changes can't break package testing
6. **Large diffs trigger full test suite:** ≥300 files → run all tests (fallback for truncated diffs from GitHub)

**VCR packages** (playback-only cassette tests):
- `libs/partners/openai`

### Python Version Matrices

Generated per-package-type:

| Package | Python Versions |
|---|---|
| `libs/core` | 3.11, 3.12, 3.13, 3.14 |
| `openai`, `perplexity` (lint/compile) | 3.11, 3.14 |
| Other packages | 3.11, 3.14 |

### Pydantic Compatibility Testing

When a package uses Pydantic:

1. Read `uv.lock` for current max version (e.g., `2.10.0` → minor = 10)
2. Read `pyproject.toml` for minimum version (e.g., `pydantic>=2.0` → minor = 0)
3. Read core's min/max Pydantic versions to filter overlaps
4. Generate test configurations for each compatible minor version:
   - Min = max(dir_min, core_min)
   - Max = min(dir_max, core_max)
   - Tests: `2.{minor}.0` for each minor in [min, max]
5. Example: if `libs/core` supports `2.0-2.10` and `libs/partners/openai` supports `2.3-2.9`, generate tests for `2.3.0`, `2.4.0`, ..., `2.9.0`

---

## Downstream Reusable Workflows

All downstream jobs use conditional execution: `if: ${{ needs.build.outputs.<job-name> != '[]' }}` to skip when no changes affect that test type.

### Reusable Workflow: `_lint.yml`

**Input parameters:**
```yaml
working-directory:  # Required: package path
python-version:     # Required: Python version to use
```

**Environment:** `ubuntu-latest`, 20 minute timeout

**Environment variables:**
```yaml
env:
  WORKDIR: ${{ inputs.working-directory == '' && '.' || inputs.working-directory }}
  RUFF_OUTPUT_FORMAT: github  # Enable inline GitHub annotations for linting failures
  UV_FROZEN: "true"
```

**Execution:**
1. Checkout & Setup Python + UV (via custom action `./.github/actions/uv_setup`)
   - Caching enabled with suffix `lint-{working-directory}`
2. Install lint & typing dependencies: `uv sync --group lint --group typing`
3. Lint package code: `make lint_package`
4. Install test dependencies (conditional):
   - Non-partners: `uv sync --inexact --group test`
   - Partners: `uv sync --inexact --group test --group test_integration`
5. Lint test code: `make lint_tests`

**Special handling:** Partners lint against both unit and integration test dependencies to catch issues in integration tests early.

---

### Reusable Workflow: `_test.yml`

**Input parameters:**
```yaml
working-directory:  # Required: package path
python-version:     # Required: Python version
```

**Environment:** `ubuntu-latest`, 20 minute timeout

**Environment variables:**
```yaml
env:
  UV_FROZEN: "true"
  UV_NO_SYNC: "true"  # Skip sync in setup; we'll sync manually in steps
```

**Execution:**
1. Checkout & Setup Python + UV
   - Cache suffix: `test-{working-directory}`
2. Install test dependencies: `uv sync --group test --dev`
3. Run unit tests with current dependencies: `make test PYTEST_EXTRA=-q`
4. **Calculate minimum dependency versions:**
   - Installs: `packaging`, `tomli`, `requests` in venv
   - Runs: `get_min_versions.py pyproject.toml pull_request {python_version}`
   - Outputs min versions as env var for next step
5. **Run tests with minimum versions** (if min versions exist):
   - Force-reinstall all dependencies at minimums: `uv pip install $MIN_VERSIONS`
   - Re-run: `make tests PYTEST_EXTRA=-q`
6. Verify clean working directory: `git status` must show "nothing to commit"

**Purpose:** Tests both current and minimum supported dependency versions to ensure backward compatibility.

---

### Reusable Workflow: `_test_pydantic.yml`

**Input parameters:**
```yaml
working-directory:    # Required: package path
python-version:       # Optional: defaults to 3.12
pydantic-version:     # Required: Pydantic version (e.g., 2.5.0)
```

**Environment:** `ubuntu-latest`, 20 minute timeout

**Execution:**
1. Checkout & Setup Python + UV
   - Cache suffix: `test-pydantic-{working-directory}`
2. Install test dependencies: `uv sync --group test`
3. Install specific Pydantic version: `uv pip install "pydantic~={pydantic-version}"`
4. Run unit tests: `make test`
5. Verify clean working directory

**Purpose:** Validates that packages work with all supported Pydantic versions (e.g., 2.0.x through 2.10.x).

---

### Reusable Workflow: `_test_vcr.yml`

**Input parameters:**
```yaml
working-directory:  # Required: package path
python-version:     # Required: Python version
```

**Environment:** `ubuntu-latest`, 20 minute timeout

**VCR cassettes** are pre-recorded HTTP interactions used in tests. Playback-only mode (no API keys):
- Runs integration tests using recorded HTTP responses
- No API keys required
- Catches test failures due to changed test inputs (would need re-recording)

**Execution:**
1. Checkout & Setup Python + UV
   - Cache suffix: `test-vcr-{working-directory}`
2. Install test dependencies: `uv sync --group test`
3. Run VCR tests in playback mode: `make test_vcr` (with `OPENAI_API_KEY=sk-fake`)
4. Verify clean working directory

**Packages with VCR tests:** `libs/partners/openai`

---

### Reusable Workflow: `_compile_integration_test.yml`

**Input parameters:**
```yaml
working-directory:  # Required: package path
python-version:     # Required: Python version
```

**Environment:** `ubuntu-latest`, 20 minute timeout

**Purpose:** Validates that integration test files compile (syntax/import checks only, no execution).

**Execution:**
1. Checkout & Setup Python + UV
   - Cache suffix: `compile-integration-tests-{working-directory}`
2. Install integration dependencies: `uv sync --group test --group test_integration`
3. Compile check: `uv run pytest -m compile tests/integration_tests`
4. Verify clean working directory

**Use case:** Catch integration test errors (missing imports, syntax errors) without running tests (faster feedback).

---

### Job: `extended-tests` (inline in check_diffs.yml)

**Environment:** `ubuntu-latest`, 20 minute timeout

**Matrix:** `${{ fromJson(needs.build.outputs.extended-tests) }}`

**Purpose:** Run extended test suites that require additional optional dependencies.

**Execution:**
```bash
echo "Running extended tests, installing dependencies with uv..."
uv venv
uv sync --group test
VIRTUAL_ENV=.venv uv pip install -r extended_testing_deps.txt
VIRTUAL_ENV=.venv make extended_tests
```

**Verification:** Ensures working directory remains clean after tests.

---

### Job: `check-release-options` (inline in check_diffs.yml)

**Environment:** `ubuntu-latest`

**Condition:** `github.repository_owner == 'langchain-ai'` (main repo only)

**Purpose:** Validates that `_release.yml` dropdown options match actual package directories on disk.

**Execution:**
```bash
python -m pip install pyyaml pytest
python -m pytest .github/scripts/test_release_options.py -v
```

**Importance:** Prevents outdated or missing options in release workflow dropdown.

---

### Job: `ci_success` — Final Status Check (inline in check_diffs.yml)

**Environment:** `ubuntu-latest`

**Depends on:** All other jobs (runs even if some fail: `always()`)

**Purpose:** Aggregate check status for branch protection rules.

**Outputs JSON:** All job results and exit code

**Merge blocking:** Returns exit code 0 (all pass) or 1 (any failure/cancellation)

---

## Integration Tests Workflow: `integration_tests.yml`

### Trigger Configuration

Two trigger modes:

**Manual trigger (workflow_dispatch):**
```yaml
inputs:
  working-directory:
    type: choice
    default: "all"
    options: [all, core, langchain, langchain_v1, text-splitters, 
              standard-tests, model-profiles, anthropic, aws, chroma, 
              deepseek, exa, fireworks, google-genai, google-vertexai, 
              groq, huggingface, mistralai, nomic, ollama, openai, 
              openrouter, perplexity, qdrant, typesafe, xai]
  working-directory-override:
    type: string
    description: "Full path override, e.g., libs/partners/partner-xyz"
  exclude:
    type: string
    description: "Comma-separated short names to drop from 'all' run"
  python-version-override:
    type: string
    description: "Single Python version override, e.g., 3.12"
```

**Scheduled trigger:**
```yaml
schedule:
  - cron: "0 13 * * *"  # Daily at 1:00 PM UTC (9 AM EDT, 6 AM PDT)
```

### Global Environment Variables

```yaml
env:
  UV_FROZEN: "true"
  DEFAULT_LIBS: >-
    ["libs/partners/openai",
    "libs/partners/anthropic",
    "libs/partners/fireworks",
    "libs/partners/groq",
    "libs/partners/mistralai",
    "libs/partners/xai",
    "libs/partners/google-vertexai",
    "libs/partners/google-genai",
    "libs/partners/aws"]
```

### Job: `compute-matrix`

**Condition:** `github.repository_owner == 'langchain-ai' || github.event_name != 'schedule'`

Prevents scheduled runs on forks (only main repo); allows manual runs from forks.

**Purpose:** Generate test matrix based on input parameters with priority hierarchy.

**Output:**
```yaml
outputs:
  matrix: JSON matrix with .include list
  python-versions: JSON array of Python versions
```

**Matrix computation logic:**

1. **Python version selection (priority order):**
   - If `python-version-override` provided: use single version (e.g., `["3.12"]`)
   - Otherwise: default `["3.11", "3.14"]`

2. **Working directory selection (priority order):**
   - If `working-directory-override` provided: use that path
   - Else if dropdown choice not `"all"`: map short name to full path:
     - Core packages: `libs/{name}` (core, langchain, langchain_v1, text-splitters, standard-tests, model-profiles)
     - Partner packages: `libs/partners/{name}` (all others)
   - Else if `exclude` specified: filter DEFAULT_LIBS
   - Otherwise: use DEFAULT_LIBS

3. **Final output:** `include` list of all `{working-directory, python-version}` pairs

**Example:** If input is `working-directory="openai"`, generates `[libs/partners/openai]` with versions `[3.11, 3.14]`.

---

### Job: `integration-tests`

**Condition:** `github.repository_owner == 'langchain-ai' || github.event_name != 'schedule'`

**Environment:** `ubuntu-latest`, 30 minute timeout

**Environment approval:** "Scheduled testing" (restricts secret access to repo members)

**Concurrency:** Per `(working-directory, python-version)` pair with **no cancellation**
```yaml
concurrency:
  group: integration-tests-${{ matrix.working-directory }}-${{ matrix.python-version }}
  cancel-in-progress: false
```

Non-cancelling concurrency prevents live API credential conflicts between manual and scheduled runs of the same package.

**Matrix:** `${{ fromJSON(needs.compute-matrix.outputs.matrix).include }}`

### Repository Checkouts

Tests exercise multiple repositories assembled into a single workspace:

```
langchain/          # Main LangChain monorepo
  libs/partners/google-genai/    (pulled from langchain-google)
  libs/partners/google-vertexai/ (pulled from langchain-google)
  libs/partners/aws/             (pulled from langchain-aws)
```

**External repos checked out separately, then reorganized:**

1. Checkout main monorepo to `./langchain`
2. Checkout `langchain-ai/langchain-google` to `./langchain-google`
3. Authenticate to Google Cloud (for VertexAI tests): `google-github-actions/auth@v3`
4. Checkout `langchain-ai/langchain-aws` to `./langchain-aws`
5. Configure AWS Credentials: `aws-actions/configure-aws-credentials@v6`
6. Reorganize external packages:
   ```bash
   rm -rf langchain/libs/partners/google-genai langchain/libs/partners/google-vertexai
   mv langchain-google/libs/genai langchain/libs/partners/google-genai
   mv langchain-google/libs/vertexai langchain/libs/partners/google-vertexai
   mv langchain-aws/libs/aws langchain/libs/partners/aws
   ```

**Dependency installation strategy:**

- **Partner packages** (in monorepo): Use `[tool.uv.sources]` in `pyproject.toml` to resolve `langchain-core`/`langchain` to local editable installs → tests current branch
- **External packages** (Google, AWS): Don't declare `[tool.uv.sources]`, so `uv sync` pulls from PyPI
  - After sync, overlay local editable installs to test against current branch:
    ```bash
    cd langchain/${{ matrix.working-directory }}
    uv sync --group test --group test_integration
    
    case "${{ matrix.working-directory }}" in
      libs/partners/google-genai)
        uv pip install -e $GITHUB_WORKSPACE/langchain/libs/core \
                       -e $GITHUB_WORKSPACE/langchain/libs/standard-tests
        ;;
      libs/partners/google-vertexai)
        uv pip install -e $GITHUB_WORKSPACE/langchain/libs/core \
                       -e $GITHUB_WORKSPACE/langchain/libs/langchain_v1 \
                       -e $GITHUB_WORKSPACE/langchain/libs/standard-tests
        ;;
      libs/partners/aws)
        uv pip install -e $GITHUB_WORKSPACE/langchain/libs/core \
                       -e $GITHUB_WORKSPACE/langchain/libs/langchain_v1 \
                       -e $GITHUB_WORKSPACE/langchain/libs/langchain \
                       -e $GITHUB_WORKSPACE/langchain/libs/standard-tests \
                       -e $GITHUB_WORKSPACE/langchain/libs/partners/anthropic
        ;;
    esac
    ```

### LangSmith Metadata Injection

Each test execution injects GitHub workflow context for tracing and observability:

```bash
metadata=$(jq -nc \
  --arg github_sha "$GH_SHA" \
  --arg github_run_id "$GH_RUN_ID" \
  --arg github_run_attempt "$GH_RUN_ATTEMPT" \
  --arg github_run_url "$GH_RUN_URL" \
  --arg github_workflow "$GH_WORKFLOW" \
  --arg github_event "$GH_EVENT" \
  --arg github_ref "$GH_REF" \
  --arg working_directory "$WORKING_DIRECTORY" \
  --arg python_version "$PYTHON_VERSION" \
  '{github_sha: $github_sha, github_run_id: $github_run_id, 
    github_run_attempt: $github_run_attempt, github_run_url: $github_run_url, 
    github_workflow: $github_workflow, github_event: $github_event, 
    github_ref: $github_ref, working_directory: $working_directory, 
    python_version: $python_version}')
echo "LANGSMITH_METADATA=$metadata" >> "$GITHUB_ENV"
```

### Environment Variables & Secrets

**LangSmith tracing configuration:**
```yaml
LANGSMITH_TRACING: "true"
LANGSMITH_API_KEY: ${{ secrets.LANGSMITH_API_KEY }}
LANGSMITH_PROJECT: ${{ vars.LANGSMITH_PROJECT || 'scheduled-testing-py' }}
LANGSMITH_TAGS: "github-actions,${working-directory},python-${python-version},sha-${github.sha}"
LANGSMITH_GATEWAY: ${{ secrets.LANGSMITH_GATEWAY }}
LANGSMITH_GATEWAY_API_KEY: ${{ secrets.LANGSMITH_GATEWAY_API_KEY }}
```

**Live API credentials** (20+ services, all injected when environment is approved):
- OpenAI (primary + Azure endpoints)
- Anthropic (+ Files API)
- Google (Vertex AI, GenAI, Search, Cloud Auth)
- Mistral, Groq, Huggingface, Fireworks, xAI, Perplexity, DeepSeek
- Ollama, OpenRouter, Together, Upstage
- AWS (+ regional config)
- EXA, Nomic, NVIDIA, Typesafe
- Astra DB, Elasticsearch, MongoDB Atlas
- WatsonX (IBM)
- AI21

**Execution:**
```bash
cd langchain/${{ matrix.working-directory }}
make integration_tests
```

**Cleanup:** After tests complete, remove external packages to avoid affecting git status check:
```bash
rm -rf langchain/libs/partners/google-genai \
       langchain/libs/partners/google-vertexai \
       langchain/libs/partners/aws
```

---

## Release Workflow: `_release.yml`

Multi-stage release pipeline with validation gates. Can be triggered as a reusable workflow (automated) or manually via `workflow_dispatch`.

### Trigger & Input Configuration

**As reusable workflow (workflow_call):**
```yaml
inputs:
  working-directory:
    type: string
    required: true
    description: "Package path, e.g., libs/core or libs/partners/openai"
  release-version:
    type: string
    default: ""
    description: "Expected version; must match pyproject.toml if provided"
  allow-prereleases:
    type: boolean
    default: false
    description: "Allow prerelease deps (unstable deps in prerelease packages)"
  skip-prior-published-package-checks:
    type: string
    default: "none"
    description: "Values: none, anthropic, openai, all (skip partner compatibility checks)"
  release-notes:
    type: string
    default: ""
    description: "Optional text prepended to auto-generated release notes"
```

**Manual trigger (workflow_dispatch):**
```yaml
inputs:
  working-directory:
    type: choice
    options: [core, langchain, langchain_v1, text-splitters, standard-tests,
              model-profiles, anthropic, aws, chroma, deepseek, exa, fireworks,
              google-genai, google-vertexai, groq, huggingface, mistralai, nomic,
              ollama, openai, openrouter, perplexity, qdrant, typesafe, xai]
  working-directory-override:
    type: string
    description: "Full path override (takes precedence)"
  release-version:
    type: string
    required: true
    default: "0.1.0"
  dangerous-nonmaster-release:
    type: boolean
    default: false
    description: "Bypass master branch check (hotfixes only)"
  [other inputs same as workflow_call]
```

### Global Environment Variables

```yaml
env:
  PYTHON_VERSION: "3.11"
  UV_FROZEN: "true"
  UV_NO_SYNC: "true"
  EFFECTIVE_WORKING_DIR: >-
    ${{ inputs.working-directory-override ||
        (startsWith(inputs.working-directory, 'libs/') && inputs.working-directory) ||
        (contains(fromJSON('["core","langchain","langchain_v1","text-splitters","standard-tests","model-profiles"]'), inputs.working-directory) && format('libs/{0}', inputs.working-directory)) ||
        format('libs/partners/{0}', inputs.working-directory) }}
```

`EFFECTIVE_WORKING_DIR` resolves dropdown short names (e.g., `"openai"`) to full paths (`"libs/partners/openai"`).

### Permissions Model

Global: `contents: read` (most jobs read-only)

Individual jobs override with:
- `id-token: write` — Trusted publishing to PyPI (OIDC)
- `contents: write` — Git tag creation for release (mark-release job only)

---

### Job: `build` — Build Distribution

**Environment:** Release (approval required)

**Condition:** `github.repository_owner == 'langchain-ai' && (github.ref == 'refs/heads/master' || inputs.dangerous-nonmaster-release)`

**Permissions:** `contents: read` (download source, not write)

**Outputs:**
```yaml
outputs:
  pkg-name: ${{ steps.check-version.outputs.pkg-name }}        # e.g., langchain-core
  version: ${{ steps.check-version.outputs.version }}          # e.g., 0.1.5
  dist-sha256sums: ${{ steps.dist-checksums.outputs.sha256sums }}  # Hash verification
```

**Execution steps:**

1. **Checkout code** (no credentials, no git push capability)
2. **Setup Python + UV** (cache disabled for security)
3. **Summarize release bypasses** (if any):
   - `dangerous-nonmaster-release`
   - `allow-prereleases`
   - `skip-prior-published-package-checks`
4. **Check version** (Python step, reads `pyproject.toml`):
   - Extract package name and version
   - Validate against input `release-version` (if provided)
   - Query PyPI: version must not already exist (prevents duplicate releases)
   - Fail closed if PyPI unreachable (security: don't risk re-publishing)
5. **Build distribution** with uv: `uv build` → `.whl` + `.tar.gz`
6. **Compute SHA256 checksums** for verification in later jobs
7. **Upload artifact** for downstream jobs

**Security:** Build job is **intentionally separate** from publish job (no shared credentials). Prevents compromised build step from accessing PyPI credentials.

---

### Job: `release-notes` — Generate Changelog

**Depends on:** `build`

**Condition:** None (runs whenever build succeeds)

**Outputs:**
```yaml
outputs:
  release-body: ${{ steps.generate-release-body.outputs.release-body }}
```

**Execution steps:**

1. **Checkout full history** (necessary for git log):
   - `fetch-depth: 0` (entire commit history)
   - Sparse checkout to package directory only
2. **Check tags** (shell script):
   - For pre-releases (version contains `-`):
     - Find latest tag matching base version (e.g., `0.1.0-rc1` → find `0.1.0` tags)
     - If not found, find latest tag of any kind
   - For regular releases:
     - Compute expected previous tag
   - Validate tag exists: `git ls-remote --exit-code --tags origin`
3. **Generate release body**:
   - Extract commits between previous tag and HEAD: `git log --format="%s" PREV_TAG..HEAD -- WORKING_DIR`
   - For first-ever release: `git rev-list --max-parents=0 HEAD`
   - Prepend optional user-provided notes

---

### Job: `pre-release-checks` — Validate Package

**Environment:** Release (approval required)

**Condition:** Depends on `build`, `release-notes`

**Timeout:** 20 minutes

**Permissions:** `contents: read`

**Cache:** Disabled (intentional: catch missing dependencies)

**Execution steps:**

1. **Checkout code** (no credentials)
2. **Setup Python + UV** (no cache)
3. **Download built wheel artifact**
4. **Import package**:
   - Create venv, install wheel
   - Python: `import <pkg>; print(dir(<pkg>))`
   - Validates package is installable and has expected exports
5. **Sync test dependencies**: `uv sync --group test`
6. **Re-install wheel** (overwrite any editable local versions)
7. **Check prerelease dependencies** (blocks if unstable):
   - Script: `check_prerelease_dependencies.py pyproject.toml`
   - Stable releases cannot depend on prerelease versions (unless `allow-prereleases=true`)
8. **Run unit tests**: `make tests` (with fresh wheel)
9. **Get minimum versions**:
   - Script: `get_min_versions.py pyproject.toml release {python_version}`
   - Parses `pyproject.toml` for minimum supported versions
10. **Test minimum versions** (if available):
    - Force-reinstall all dependencies at minimums: `uv pip install $MIN_VERSIONS`
    - Re-run tests: `make tests PYTEST_EXTRA="-q -k 'not test_serdes'"`
11. **Import integration test dependencies** (partner packages only):
    - `uv sync --group test --group test_integration`
12. **Run integration tests** (partner packages only):
    - `make integration_tests`
    - Uses live API credentials (secrets injected)

**API credentials injected** (partner packages):
- LangSmith, OpenAI (+ Azure), Anthropic (+ Files API), Google (Vertex AI, GenAI, Search), Mistral, Groq, Huggingface, EXA, Nomic, WatsonX, Astra DB, Elasticsearch, MongoDB, Upstage, Fireworks, xAI, DeepSeek, Ollama, OpenRouter, and others

---

### Job: `test-pypi-publish` — Publish to Test PyPI

**Environment:** Release (approval required)

**Depends on:** `build`, `release-notes`, `pre-release-checks`

**Permissions:** `id-token: write` (trusted publishing via OIDC, no password needed)

**Execution steps:**

1. Checkout code (no credentials)
2. Download wheel artifact
3. **Verify checksums** (fail if mismatch)
4. **Publish to test.pypi.org**:
   - Repository URL: `https://test.pypi.org/legacy/`
   - `skip-existing: true` (idempotent: re-runs don't fail if already published)
   - Uses `pypa/gh-action-pypi-publish@v1`

**Purpose:** Validates PyPI upload process before publishing to production. Catches authentication or metadata issues early.

---

### Job: `test-prior-published-packages-against-new-core`

**Condition:** Only runs when releasing `libs/core`

**Depends on:** `build`, `release-notes`, `test-pypi-publish`, `pre-release-checks`

**Matrix:** Partners: `anthropic`, `openai`

**Purpose:** For core releases, validate that latest published partner packages still work with the new core.

**Per-partner execution steps:**

1. **Checkout LangChain** (no credentials)
2. **Check if libs/core** (exit early if not)
3. **Setup Python + UV** (no cache)
4. **Download new core artifact**
5. **Check skip condition** (exit if `skip-prior-published-package-checks=none` or includes partner)
6. **Query PyPI** for latest published partner:
   - Find latest non-yanked final release (PEP 440)
   - Fail closed if PyPI unreachable
7. **Verify git tag exists**: Partner release must have a git tag
8. **Fetch and checkout published code**:
   - Shallow fetch the partner tag
   - Remove current partner code, restore from fetched tag
9. **Install and test against new core**:
   - `uv sync --group test --group test_integration`
   - `uv pip install ../../core/dist/*.whl` (override with new core)
   - `make test`
   - `make integration_tests` (with live API keys)

---

### Job: `test-dependents` — External Project Testing

**Condition:** Only runs when releasing `libs/core` or `libs/langchain_v1`

**Depends on:** `build`, `release-notes`, `test-pypi-publish`, `pre-release-checks`

**Matrix:**
- Python: `[3.11, 3.13]`
- Projects: `deepagents` (from `langchain-ai/deepagents`)

**Purpose:** Validate that external projects (not in monorepo) still work with new core/langchain_v1.

**Execution steps:**

1. **Checkout LangChain** (no credentials)
2. **Checkout external project** (e.g., deepagents)
3. **Setup Python + UV** (no cache)
4. **Download artifact**
5. **Install with overrides**:
   - `uv sync --group test`
   - `uv pip install dist/*.whl` (override external's pinned versions)
6. **Run tests**: `make test`

---

### Job: `publish` — Publish to PyPI

**Condition:** `!cancelled() && !failure()` (run unless explicitly cancelled or upstream failure)

**Environment:** Release (approval required)

**Depends on:** `build`, `release-notes`, `test-pypi-publish`, `pre-release-checks`, `test-dependents`, `test-prior-published-packages-against-new-core`

**Permissions:** `id-token: write` (trusted publishing)

**Execution steps:**

1. Checkout code (no credentials)
2. Setup Python + UV (no cache)
3. Download wheel artifact
4. **Verify checksums** (fail if mismatch)
5. **Publish to pypi.org**:
   - Uses `pypa/gh-action-pypi-publish@v1`
   - Verbose output and hash logging
   - Trusted publishing via OIDC (no password)

---

### Job: `mark-release` — Create Git Tag & GitHub Release

**Condition:** `!cancelled() && !failure()`

**Environment:** Release (approval required)

**Permissions:** `contents: write` (git tag creation, GitHub release)

**Depends on:** `build`, `release-notes`, `test-pypi-publish`, `pre-release-checks`, `publish`

**Execution steps:**

1. Checkout code (no credentials initially)
2. Setup Python + UV (no cache)
3. Download artifact
4. **Create release** (via `ncipollo/release-action@v1`):
   - **Tag name format:** `{pkg-name}=={version}` (e.g., `langchain-core==0.1.5`)
   - **Release body:** Auto-generated changelog from `release-notes` job
   - **Artifacts:** Attach `.whl` and `.tar.gz` to release
   - **Latest flag:** `makeLatest: true` only for `langchain-core` (primary package)

---

## Supporting Utilities & Actions

### Custom Action: `.github/actions/uv_setup`

**Purpose:** Unified Python + UV setup with intelligent caching.

**Inputs:**
```yaml
python-version:      # Required: e.g., 3.11, 3.12
enable-cache:        # Optional: "true" (default) or "false" (for release security)
cache-suffix:        # Optional: suffix for cache key uniqueness
working-directory:   # Optional: glob scope for cache dependencies
```

**Implementation:**
- Uses `astral-sh/setup-uv@v7` (official uv action)
- UV version pinned: `0.13.6`
- Cache glob patterns:
  - `{working-directory}/pyproject.toml`
  - `{working-directory}/uv.lock`
  - `{working-directory}/requirements*.txt`
- Cache suffix enables per-job caching (lint, test, release all have separate caches)

**Security:** Cache disabled in release jobs (prevents stale dependency versions from bypassing checks).

---

## Environment Variables Reference

### Global CI Environment Variables

| Variable | Value | Purpose |
|---|---|---|
| `UV_FROZEN` | `"true"` | Enforce frozen `uv.lock`; no dependency resolution updates |
| `UV_NO_SYNC` | `"true"` (check_diffs) | Skip expensive sync in build job (only analyzes files) |
| `RUFF_OUTPUT_FORMAT` | `github` | Inline GitHub annotations for linting failures |

### Dynamic Matrices

Matrices are JSON arrays output by the `build` job and consumed by downstream jobs:

```json
[
  {"working-directory": "libs/core", "python-version": "3.11"},
  {"working-directory": "libs/core", "python-version": "3.12"},
  ...
]
```

Matrix jobs expand with:
```yaml
strategy:
  matrix:
    job-configs: ${{ fromJson(needs.build.outputs.lint) }}
with:
  working-directory: ${{ matrix.job-configs.working-directory }}
  python-version: ${{ matrix.job-configs.python-version }}
```

---

## Secret Management

### Scope: "Scheduled testing" Environment

The `integration_tests.yml` workflow requires approval from "Scheduled testing" environment. This restricts secrets to workflow runs that have been reviewed:

- Scheduled runs must complete if `github.repository_owner == 'langchain-ai'`
- Manual (workflow_dispatch) runs always proceed (fork-friendly)

### Secrets Used in Integration Tests

All secrets below are injected into the `integration-tests` job environment:

**Tracing & Observability:**
- `LANGSMITH_API_KEY`
- `LANGSMITH_GATEWAY`
- `LANGSMITH_GATEWAY_API_KEY`

**Model Provider APIs:**
- `OPENAI_API_KEY`, `AZURE_OPENAI_*` (multiple)
- `ANTHROPIC_FILES_API_*` (image/PDF IDs)
- `GOOGLE_API_KEY`, `GOOGLE_SEARCH_API_KEY`, `GOOGLE_CSE_ID`, `GOOGLE_CREDENTIALS`
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`
- `GROQ_API_KEY`, `MISTRAL_API_KEY`, `XAI_API_KEY`, `DEEPSEEK_API_KEY`
- `COHERE_API_KEY`, `HUGGINGFACEHUB_API_TOKEN`, `FIREWORKS_API_KEY`, `UPSTAGE_API_KEY`
- `WATSONX_APIKEY`, `WATSONX_PROJECT_ID`

**Vector Databases & Data Stores:**
- `ASTRA_DB_API_ENDPOINT`, `ASTRA_DB_APPLICATION_TOKEN`, `ASTRA_DB_KEYSPACE`
- `ELASTICSEARCH_*` (ES_URL, ES_CLOUD_ID, ES_API_KEY)
- `MONGODB_ATLAS_URI`, `QDRANT_*`

**Specialized Services:**
- `EXA_API_KEY`, `NOMIC_API_KEY`, `NVIDIA_API_KEY`, `OLLAMA_API_KEY`
- `OPENROUTER_API_KEY`, `TOGETHER_API_KEY`, `TYPESAFE_API_KEY`, `PPLX_API_KEY`
- `LANGCHAIN_TESTS_USER_AGENT`

**Release Environment (Release approval required):**
- Trusted publishing via OIDC (no stored PyPI token)
- `GITHUB_TOKEN` (automatic, for git operations)

---

## Conditional Execution Patterns

### Matrix Conditional Skips

```yaml
if: ${{ needs.build.outputs.lint != '[]' }}  # Skip if no changes affected this test type
```

When change detection produces empty matrices (`[]`), downstream jobs skip automatically.

### Environment-Based Conditionals

```yaml
if: github.repository_owner == 'langchain-ai'  # Skip on forks
```

Prevents scheduled or credential-requiring jobs from running on fork PRs.

### Job Dependency Flow

```yaml
needs: [build]
if: ${{ !contains(github.event.pull_request.labels.*.name, 'ci-ignore') }}
```

Multiple conditions can combine with `&&` (AND) or `||` (OR).

### Concurrency Control

```yaml
concurrency:
  group: integration-tests-${{ matrix.working-directory }}-${{ matrix.python-version }}
  cancel-in-progress: false  # Serialize same-package runs
```

- `cancel-in-progress: true` (check_diffs) — Cancel earlier runs to save resources
- `cancel-in-progress: false` (integration_tests) — Serialize to prevent live API conflicts

---

## Best Practices & Invariants

### Change Detection Robustness

- Large diffs (≥300 files) trigger full test suite (GitHub diff truncation fallback)
- Infrastructure changes trigger extended tests on all core packages (conservative safety)
- Dependency graph ensures all affected packages are tested, not just directly changed packages

### Test Coverage Strategy

- **Unit tests**: All Python versions (3.11-3.14 for core)
- **Minimum versions**: Every release tests against minimum supported dependencies
- **Pydantic versions**: Core and Pydantic-dependent packages test all compatible minor versions
- **VCR playback**: Catches test input changes without re-recording
- **Integration compilation**: Detects import/syntax errors early (no API key cost)
- **Extended tests**: Optional dependencies and advanced features

### Security Practices

- **Build isolation**: Build job has no publish credentials
- **Trusted publishing**: PyPI authentication via OIDC, no stored tokens
- **Permission minimization**: Jobs declare minimal required permissions
- **Cache security**: Cache disabled in release jobs (prevent dependency bypass)
- **Secret scoping**: Integration tests restricted to "Scheduled testing" environment approval
- **Fail-closed design**: PyPI unreachable → abort release (don't risk duplicate)

### Resource Optimization

- **Concurrency cancellation**: Redundant PR runs cancelled (check_diffs only)
- **Dynamic matrices**: Only test affected packages
- **Conditional steps**: Skip installation if not needed
- **Cache efficiency**: Per-job cache suffixes prevent conflicts

---

## Troubleshooting Common Issues

### CI Bypass Label

Use `ci-ignore` label on PR to skip the `build` job entirely. All downstream jobs will be skipped.

### Dependency Version Issues

- **Minimum version failures**: Run `get_min_versions.py` locally to debug
- **Pydantic incompatibility**: Check Pydantic version matrix generation in `check_diff.py`
- **Prerelease blocking**: Set `allow-prereleases=true` in release if intentional

### Integration Test Credentials

- Scheduled runs only proceed if `github.repository_owner == 'langchain-ai'`
- Manual runs on forks will fail if they require approved secrets
- LangSmith metadata injection requires `LANGSMITH_API_KEY` and `LANGSMITH_PROJECT`

### Release Workflow Failures

- **Version already published**: Check PyPI; bump version in `pyproject.toml`
- **Git tag conflict**: Ensure tag format is unique (use `langchain-core==0.1.5`)
- **Checksum mismatch**: Rebuild artifacts (clean build dir)
