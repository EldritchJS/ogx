# OGX Codebase Overview

**Generated:** 2026-05-26  
**Purpose:** Comprehensive exploration of the OGX codebase architecture, structure, and implementation

---

## Executive Summary

OGX (formerly Llama Stack) is an **OpenAI-compatible API server** with a pluggable provider architecture. It's a drop-in replacement for the OpenAI API that works with any model (Llama, GPT, Gemini, Mistral, etc.) on any infrastructure (Ollama, vLLM, cloud providers). The core value proposition: write once against the OpenAI API, swap backends without code changes.

**Key Stats:**
- ~450 Python files, 71K+ lines of code
- 306 test files with record/replay system
- 32 GitHub workflows
- 19+ inference providers supported
- Python 3.12 required, `uv` for dependency management

---

## Repository Structure

### Package Organization

```text
src/
├── ogx_api/          # Lightweight API protocol definitions (separate PyPI package)
│   ├── inference/    # Inference protocol, models, FastAPI routes
│   ├── responses/    # Responses API protocol and routes
│   ├── messages/     # Anthropic Messages API
│   ├── interactions/ # Google GenAI Interactions API
│   ├── vector_io/    # Vector store operations
│   ├── files/        # File management
│   ├── batches/      # Batch processing
│   ├── providers/    # Provider spec types
│   └── internal/     # KVStore/SqlStore interfaces
│
├── ogx/              # Server implementation (~70% of codebase)
│   ├── core/         # Server core: routing, resolution, storage
│   ├── providers/    # Provider implementations
│   ├── distributions/# Pre-built configs for different environments
│   ├── cli/          # CLI commands (ogx stack run, build, etc.)
│   ├── testing/      # Test infrastructure (api_recorder)
│   └── telemetry/    # OpenTelemetry integration
│
└── ogx_ui/           # Next.js web UI for chat playground and admin
    ├── app/          # Next.js app router pages
    └── components/   # React components
```

### Core Directories Deep Dive

**`src/ogx/core/`** - Server implementation:
```text
core/
├── server/              # FastAPI server, auth middleware, quota middleware
├── routers/             # API-specific routers (inference, responses, vector_io)
├── routing_tables/      # Resource-to-provider mapping tables
├── storage/             # KVStore and SqlStore backends
│   ├── kvstore/         # Key-value store (SQLite, Redis, PostgreSQL, MongoDB)
│   └── sqlstore/        # SQL store (SQLite, PostgreSQL)
├── store/               # Distribution registry (persists registered resources)
├── access_control/      # Access control policy enforcement
├── conversations/       # Conversation service (persistence for chat threads)
├── prompts/             # Prompt service (prompt template management)
├── utils/               # Config resolution, context propagation, dynamic import
├── resolver.py          # Provider resolution engine: validate, sort, instantiate
├── distribution.py      # Provider registry loading, API enumeration
├── stack.py             # Stack class: initialization, resource registration, lifecycle
└── datatypes.py         # Core data types (StackConfig, Provider, etc.)
```

**`src/ogx/providers/`** - All provider implementations:
```text
providers/
├── inline/              # In-process providers (run locally)
│   ├── inference/       # Built-in inference, meta-reference
│   ├── responses/       # Responses orchestration
│   ├── vector_io/       # ChromaDB, FAISS, PGVector, SQLite-vec, Weaviate, Elasticsearch
│   ├── tool_runtime/    # Tool execution runtime
│   ├── file_processor/  # Document processing
│   └── batches/         # Batch processing
│
├── remote/              # Remote service adapters (call external APIs)
│   ├── inference/       # 19+ providers (see below)
│   ├── agents/          # Remote agent providers
│   ├── vector_io/       # Remote vector stores
│   ├── tool_runtime/    # Remote tool execution
│   └── file_processor/  # Remote file processing
│
├── registry/            # Provider spec declarations (available_providers())
│   ├── inference.py
│   ├── responses.py
│   ├── vector_io.py
│   └── ...
│
└── utils/               # Shared provider utilities
    ├── inference/       # OpenAI mixin, model registry
    └── ...
```

---

## Architecture

### Request Flow

```text
Client (ogx-client SDK or raw HTTP)
  ↓
FastAPI Server  (src/ogx/core/server/server.py)
  ├── AuthenticationMiddleware  (token validation, user extraction)
  ↓
Route Dispatch
  ├── FastAPI Router routes     (auto-discovered via fastapi_router_registry.py)
  ↓
Router  (src/ogx/core/routers/)
  ├── Looks up resource (model, vector store, tool group) in RoutingTable
  ├── Resolves which provider handles this resource
  ├── Enforces access control policies
  ↓
Provider Implementation
  ├── Inline provider (runs in-process, e.g. meta-reference, sqlite-vec)
  ├── Remote provider (calls external service, e.g. ollama, openai, fireworks)
  ↓
External Service or Local Computation
```

### Example: Chat Completion Request

1. Client sends `POST /v1/chat/completions` with `model: "ollama/llama3.2:3b-instruct-fp16"`
2. `server.py` dispatches to the inference FastAPI router
3. `InferenceRouter` calls `routing_table.get_provider_impl(model_id)`
4. `CommonRoutingTableImpl` looks up model in `DistributionRegistry`, finds provider `ollama`
5. Router delegates to `ollama` provider's `openai_chat_completion()` method
6. Ollama provider (extends `OpenAIMixin`) creates `AsyncOpenAI` client → Ollama server
7. Response streams back through router to client as SSE events

### Provider Architecture

```text
Provider Types:
  ├── InlineProviderSpec    (runs in-process)
  │     provider_type: "inline::builtin"
  │     module: "ogx.providers.inline.inference.builtin"
  │
  └── RemoteProviderSpec    (adapts external service)
        provider_type: "remote::ollama"
        module: "ogx.providers.remote.inference.ollama"
```

**Provider Spec Declaration:**
Each provider spec includes:
- `api` - which API it implements (e.g., `Api.inference`)
- `provider_type` - unique identifier like `"remote::openai"`
- `module` - Python module with factory function
- `config_class` - Pydantic config model
- `pip_packages` - runtime dependencies
- `deprecation_warning` - (optional) for deprecated providers
- `toolgroup_id` - (optional) for tool_runtime providers

**Provider Registration Flow:**
1. Each API has a file in `registry/` (e.g., `registry/inference.py`)
2. File exports `available_providers()` returning list of `ProviderSpec` objects
3. At startup, `get_provider_registry()` in `core/distribution.py` collects all specs
4. `resolve_impls()` in `core/resolver.py`:
   - Validates providers against registry
   - Sorts by dependency order
   - Instantiates each provider via factory function

### Auto-Routing System

Many APIs use automatic routing where multiple providers can serve the same API:

| Routing Table API   | Router API         | Purpose                                    |
|---------------------|--------------------|--------------------------------------------|
| `Api.models`        | `Api.inference`    | Multiple providers serve different models  |
| `Api.tool_groups`   | `Api.tool_runtime` | Route tool calls to correct provider       |
| `Api.vector_stores` | `Api.vector_io`    | Route vector operations to correct backend |

The routing table tracks which provider owns which resource, and the router delegates requests accordingly.

---

## API Coverage

### OpenAI API Compatibility

OGX implements the OpenAI API with high conformance:

**Core APIs:**
- `/v1/chat/completions` - Chat completions (streaming & non-streaming)
- `/v1/completions` - Legacy completions
- `/v1/embeddings` - Text embeddings
- `/v1/files` - File upload and management
- `/v1/vector_stores` - Vector store CRUD
- `/v1/batches` - Offline batch processing

**Responses API (OpenAI Assistants-style):**
- `/v1/responses` - Server-side agentic orchestration
- Tool calling with MCP server integration
- Built-in file search (RAG)
- **Open Responses conformant** - passes conformance test suite

### Multi-SDK Support

OGX natively supports multiple SDKs beyond OpenAI:

1. **Anthropic SDK** - `/v1/messages` endpoint (Claude Messages API)
2. **Google GenAI SDK** - `/v1alpha/interactions` endpoint (Gemini API)
3. **OpenAI SDK** - Full `/v1/*` compatibility

You can use all three SDKs against the same server simultaneously, regardless of which backend provider is actually running the models.

---

## Provider Ecosystem

### Remote Inference Providers (19+)

All providers in `src/ogx/providers/remote/inference/`:

```
anthropic           Azure (OpenAI)      Bedrock (AWS)
Cerebras            Databricks          Fireworks
Gemini (Google)     Groq                Llama.cpp Server
Nvidia              OCI (Oracle)        Ollama
OpenAI              Passthrough         Runpod
Sambanova           Together AI         vLLM
...and more
```

Each provider adapter:
- Extends `OpenAIMixin` for consistent OpenAI-compatible behavior
- Implements the `Inference` protocol from `ogx_api`
- Has a Pydantic config class for validation
- Declares dependencies and required packages

### Inline Providers

**Inference:**
- `builtin` - Meta-reference implementation
- `meta-reference` - Reference inference for testing

**Vector Stores:**
- ChromaDB Client
- Elasticsearch
- FAISS (CPU)
- PGVector (PostgreSQL)
- SQLite-vec
- Weaviate

**Other Inline Providers:**
- File processors (document parsing, chunking)
- Tool runtime (MCP integration, function execution)
- Batch processing
- Responses orchestration
- iOS-specific integrations

### Distributions

Pre-built configurations for different environments (`src/ogx/distributions/`):

| Distribution      | Purpose                                      |
|-------------------|----------------------------------------------|
| `starter`         | Default dev setup with Ollama                |
| `nvidia`          | NVIDIA GPU-optimized stack                   |
| `watsonx`         | IBM WatsonX integration                      |
| `oci`             | Oracle Cloud Infrastructure                  |
| `ci-tests`        | Test environment configuration               |
| `postgres-demo`   | PostgreSQL storage backend demo              |
| `open-benchmark`  | Benchmarking configuration                   |

Each distribution is a YAML config bundling specific providers for a target environment.

---

## Storage Architecture

### Dual-Store Design

OGX uses two types of storage backends:

#### 1. KVStore (Key-Value Storage)

**Interface:** `src/ogx/core/storage/kvstore/`

| Backend    | Config Class             | Use Case                    |
|------------|--------------------------|-----------------------------|
| SQLite     | `SqliteKVStoreConfig`    | Default, single-node        |
| Redis      | `RedisKVStoreConfig`     | Multi-node, caching         |
| PostgreSQL | `PostgresKVStoreConfig`  | Production deployments      |
| MongoDB    | `MongoDBKVStoreConfig`   | Document-oriented workloads |

**Used for:**
- Distribution registry (persists registered resources)
- Quota tracking
- Provider state management

#### 2. SqlStore (SQL Storage)

**Interface:** `src/ogx/core/storage/sqlstore/`

| Backend    | Config Class             | Use Case                |
|------------|--------------------------|-------------------------|
| SQLite     | `SqliteSqlStoreConfig`   | Default, single-node    |
| PostgreSQL | `PostgresSqlStoreConfig` | Production deployments  |

**Used for:**
- Inference store (chat completion logs)
- Conversations (chat thread persistence)
- Prompts (template management)

### Storage Configuration

Defined in the `storage` section of run config:

```yaml
storage:
  type: sqlite
  db_path: ${env.SQLITE_STORE_DIR}/registry.db
  stores:
    kvstore:
      type: kv_sqlite
      db_path: ${env.SQLITE_STORE_DIR}/kvstore.db
    inference:
      type: sql_sqlite
      db_path: ${env.SQLITE_STORE_DIR}/inference_store.db
```

Environment variable substitution: `${env.VAR_NAME:=default_value}`

---

## Configuration System

### Run Config (StackConfig)

A YAML file defining everything about a running OGX instance:

```yaml
version: 2
distro_name: starter
apis:
  - inference
  - responses
  - vector_io
  # ...
providers:
  inference:
    - provider_id: ollama
      provider_type: remote::ollama
      config:
        base_url: ${env.OLLAMA_URL:=http://localhost:11434/v1}
    - provider_id: openai
      provider_type: remote::openai
      config:
        api_key: ${env.OPENAI_API_KEY}
storage:
  type: sqlite
  db_path: ~/.ogx/storage/registry.db
```

**Key Features:**
- **Environment variable substitution:** `${env.VAR_NAME:=default}`
- **Conditional providers:** `${env.API_KEY:+provider_id}` enables provider only when env var set
- **Multiple providers per API:** Both Ollama and OpenAI can serve inference, each handling different models

### Build Config

Used by `ogx build` to create container images. Declares:
- Which providers to include
- What packages to install
- Distribution-specific dependencies

Versioned separately from run config.

---

## Testing Infrastructure

### Record/Replay System

**Location:** `src/ogx/testing/api_recorder.py`

Sophisticated integration test system that intercepts OpenAI client calls:

**How it works:**
1. **Recording Mode:** Tests run against real server. `APIRecorder` monkey-patches (runtime modification of code behavior—replacing methods on the `OpenAI` client class at runtime to intercept calls) `OpenAI` client methods to capture every request/response pair
2. **Storage:** Responses stored as JSON files under `tests/integration/recordings/` organized by provider and test
3. **Replay Mode:** In CI, recorder matches incoming requests to stored responses by hashing request parameters
4. **Deterministic IDs:** Recorder overrides ID generation during replay for reproducibility

**Modes** (controlled by `--inference-mode` or `OGX_TEST_INFERENCE_MODE`):
- `replay` (default) - Use cached responses, no API keys needed
- `record` - Force-record all interactions
- `record-if-missing` - Record only when no cached response exists
- `live` - Bypass recording, make real calls

**Benefits:**
- Fast CI runs (no real API calls)
- Deterministic tests
- No API keys required for CI
- Easy to update when APIs change

### Test Organization

```text
tests/
├── unit/                     # Fast, isolated tests
├── integration/              # End-to-end tests with record/replay
│   ├── recordings/           # Cached API responses (JSON)
│   ├── responses/            # Responses API tests
│   ├── vision/               # Vision model tests
│   └── base/                 # Base inference tests
└── README.md
```

**Running Tests:**

```bash
# Unit tests
./scripts/unit-tests.sh

# Integration tests (replay mode - default)
uv run ./scripts/integration-tests.sh \
  --stack-config server:ci-tests --setup gpt \
  --suite responses

# Re-record missing recordings (requires OPENAI_API_KEY)
uv run ./scripts/integration-tests.sh \
  --stack-config server:ci-tests --setup gpt \
  --inference-mode record-if-missing \
  --file tests/integration/responses/test_compact_responses.py
```

---

## Development Guidelines

### Required Tooling

- **Python 3.12** - Pre-commit hooks only work with 3.12
- **`uv`** - All dependency management and script execution
  - ✅ `uv run pytest tests/unit/`
  - ❌ `python3 -m pytest tests/unit/`

### Code Style Requirements

**Type Hints:**
- All function signatures must have type hints
- Prefer Pydantic models for validation
- Code must pass `mypy` (check exclude list in `pyproject.toml`)

**Logging:**
- Use `from ogx.log import get_logger`
- Always use key-value style: `logger.info("Processing", model=model_id)`
- ❌ NO f-strings or %-formatting in logs (enforced by pre-commit hook)
- Error messages must start with "Failed to ..."

**Comments:**
- Comments must add value, not describe what the code does
- Only clarify surprising behavior that's non-obvious
- Good variable naming > comments
- ❌ Do NOT remove existing comments unless factually wrong

**Functions:**
- Use `def _function_name` for private functions
- Prefer explicit top-level imports over inline imports
- Do not use exceptions as control flow

### Git Conventions

**Commits:**
- Always use `--signoff` (`-s`) when creating commits
- Use [conventional commits](https://www.conventionalcommits.org/) format
- Full sentences in commit messages, not bullet points
- ❌ Do NOT amend commits during PR review - use new commits
- ✅ Use `git merge main` to update branches, not `git rebase`

**Before Pushing:**
- Merge `upstream/main` to avoid CI failures from stale code
- Run pre-commit checks: `uv run pre-commit run --all-files`

### API Changes

When modifying or extending APIs:

1. Update models in `src/ogx_api/`
2. Regenerate OpenAPI specs: `uv run ./scripts/run_openapi_generator.sh`
3. Check for breaking changes (enforced by pre-commit hook)
4. Include test plan with execution output in PR description

**API Stability Contract:**
- HTTP API levels: `/v1alpha`, `/v1beta`, `/v1`
- Two `ogx-api` surfaces: `ogx_api.types`, `ogx_api.provider`
- Read `docs/docs/concepts/apis/api_leveling.mdx` before:
  - Adding a new API
  - Graduating API between levels
  - Removing/renaming field on `v1` datatype
  - Changing on-disk storage schemas

### Adding a New Provider

1. Create directory under `inline/` or `remote/` for appropriate API
2. Implement the API protocol from `ogx_api`
3. Add `ProviderSpec` entry to corresponding file in `registry/`
4. Run codegen: `uv run ./scripts/provider_codegen.py`
5. Update distribution configs if needed

**Search for existing patterns first:**
```bash
# Find examples of deprecation, validation, etc.
grep -r "deprecation_warning" src/ogx/providers/registry/
```

---

## Recent Development Activity

### Latest Commits (Last 15)

Recent work focuses on:
- **Responses API conformance** - Schema transforms for Open Responses compliance
- **New CLI features** - `ogx connect claude` command
- **Security fixes** - Authorization checks, client lifecycle management
- **Multi-SDK support** - Anthropic and Google SDK examples
- **Error handling** - Proper 4xx propagation instead of 500 errors
- **Documentation** - RAG benchmarks blog post
- **Dependency updates** - Docker actions, CodeQL, Python packages with CVE fixes

### Active Areas

Based on commit history and code structure:

1. **Responses API** - Heavy development on agentic orchestration
2. **Multi-SDK** - Expanding beyond OpenAI to Anthropic and Google
3. **Security** - Authorization, CVE tracking, client lifecycle
4. **Performance** - Client reuse, bounded reads, resource cleanup
5. **Documentation** - Blogs, examples, API docs

---

## Key Classes & Entry Points

| Component | Location | Purpose |
|-----------|----------|---------|
| `OGX` | `core/stack.py` | Composite class implementing all API protocols |
| `Stack` | `core/stack.py` | Initialization, resource registration, lifecycle |
| `StackApp` | `core/server/server.py` | FastAPI app wrapper |
| `resolve_impls()` | `core/resolver.py` | Provider instantiation and dependency resolution |
| `CommonRoutingTableImpl` | `core/routing_tables/common.py` | Base routing table for all auto-routed APIs |
| `InferenceRouter` | `core/routers/inference.py` | Routes inference calls to correct provider |
| `OpenAIMixin` | `providers/utils/inference/openai_mixin.py` | Shared OpenAI-compatible client logic |
| `get_provider_registry()` | `core/distribution.py` | Loads all available provider specs |
| `APIRecorder` | `testing/api_recorder.py` | Record/replay test infrastructure |
| `DistributionRegistry` | `core/store/` | Persistent registry of resources |

---

## Documentation

### Available Documentation

**Root-level docs:**
- `README.md` - Quick start & architecture diagram
- `ARCHITECTURE.md` - System architecture deep dive
- `AGENTS.md` - AI agent guidelines for codebase work
- `CONTRIBUTING.md` - Contribution guidelines
- `RELEASE_PROCESS.md` - Release workflow
- `SECURITY.md` - Security policy

**Module-level READMEs:**
Every major directory has a README explaining its purpose:
- `src/ogx/README.md`, `src/ogx/core/README.md`
- `src/ogx/providers/README.md`, `src/ogx/providers/inline/README.md`
- `src/ogx/core/server/README.md`, `src/ogx/core/storage/README.md`
- ...and many more

**Docusaurus Site:**
- Full documentation site at `docs/`
- Notebooks, tutorials, API references
- Blog posts (including recent RAG benchmarks)
- Provider documentation (auto-generated from configs)

### Updating Documentation

**When making code changes, check if these need updating:**
- Root README architecture diagram (when adding/removing providers or APIs)
- `ARCHITECTURE.md` (system changes)
- Module-level READMEs (when adding/renaming modules)
- Provider docs are auto-generated via: `uv run ./scripts/provider_codegen.py`
- OpenAPI specs: `uv run ./scripts/run_openapi_generator.sh`

---

## Code Quality & Safety

### Security Measures

**CVE Tracking:**
`pyproject.toml` maintains version constraints for known vulnerabilities:
```toml
[tool.uv]
constraint-dependencies = [
    "aiohttp>=3.13.4",      # CVE-2026-34514 + 9 more
    "authlib>=1.6.11",      # CVE-2026-41425 + 7 more
    "cryptography>=46.0.7", # CVE-2026-39892, CVE-2026-34073
    # ... dozens more
]
```

**Pre-commit Hooks:**
- API spec breaking change detection
- Logging format enforcement (no f-strings)
- `ogx.log` usage enforcement
- Linting, formatting, type checking
- Markdown linting
- YAML validation

**Best Practices:**
- Avoid command injection, XSS, SQL injection, OWASP top 10
- Validate at system boundaries only (user input, external APIs)
- Trust internal code and framework guarantees
- No premature error handling for scenarios that can't happen

### Code Standards

**Comments:**
- Default to writing NO comments
- Only add comments for non-obvious WHY (hidden constraints, subtle invariants, workarounds)
- Never explain WHAT the code does (well-named identifiers do that)
- Don't reference current task, fix, or callers (belongs in PR description)

**Code Style:**
- No features/refactoring beyond task requirements
- Three similar lines > premature abstraction
- No error handling for scenarios that can't happen
- No feature flags or backwards-compatibility shims when you can just change the code
- No backwards-compatibility hacks for unused code (delete it completely)

---

## Interesting Patterns & Design Decisions

### 1. Declarative Provider Registry

Providers are discovered at startup via `available_providers()` functions. No manual registration needed.

**Why this is interesting:** Traditional plugin systems require explicit registration (calling `register_provider(MyProvider)` somewhere), which creates tight coupling and makes it easy for the registry to drift out of sync with available implementations. OGX's approach uses Python's import system for discovery—each registry file (`registry/inference.py`, etc.) exports an `available_providers()` function that returns specs. The server imports these at startup and gets a complete, accurate list automatically. Adding a new provider means creating the implementation and adding one entry to the registry function—no hunt for a central registration file, no risk of forgetting to register. This also makes it trivial for third-party packages to extend OGX: just provide your own registry module.

### 2. Environment Variable Substitution in Config

```yaml
base_url: ${env.OLLAMA_URL:=http://localhost:11434/v1}
api_key: ${env.OPENAI_API_KEY}
conditional_provider: ${env.API_KEY:+provider_id}  # Only if env var set
```

**Why this is interesting:** This solves multiple problems at once. First, it's 12-factor compliant—configuration lives in the environment, not checked into git. Second, the `:=default` syntax means the same config works locally (uses default) and in prod (uses env var) without maintaining separate files. Third, the `:+` syntax enables conditional provider loading: the provider only activates if you've set the API key, so you can ship a "batteries included" distribution with 20 providers defined, but only the ones you've configured actually start. This means one distribution config can serve development (Ollama only), staging (Ollama + OpenAI), and production (all providers) just by setting different env vars. No config duplication, no drift.

### 3. Multi-Tenant Auto-Routing

Same API endpoint serves multiple providers based on resource identifier:
- `ollama/llama3.2:3b` → Ollama provider
- `gpt-4` → OpenAI provider
- `claude-3-opus` → Anthropic provider

All through `/v1/chat/completions`.

**Why this is interesting:** Most multi-backend systems require different endpoints per backend (e.g., `/ollama/v1/...` vs `/openai/v1/...`) or require clients to specify the backend explicitly. OGX's routing table parses the model ID at request time and dispatches to the correct provider transparently. This means client code is completely backend-agnostic—you can swap `gpt-4` for `llama-3.3-70b` in one place (where you set the model string) without changing anything else. It also enables gradual migrations: during a provider switch, both old and new model IDs work simultaneously, routed to different backends, with no client changes. The routing decision happens server-side based on declarative mappings in the distribution config, so teams get multi-provider support without any client-side orchestration logic.

### 4. Codegen for Documentation

Provider configuration classes auto-generate documentation:
- Pydantic `Field` with `description` parameters
- Run `scripts/provider_codegen.py` to update docs
- Never edit generated provider docs manually

**Why this is interesting:** Solves the perennial documentation drift problem. The source of truth is the actual Pydantic config class used at runtime—developers add `description=` to fields when writing provider code, and those descriptions become the official documentation. There's no separate docs file to update, no risk of docs describing fields that don't exist or having the wrong types. When you run codegen, it introspects every provider's config schema and generates markdown docs automatically. This is a forcing function for good config design (if you can't describe a field concisely, maybe it shouldn't exist) and guarantees that documented configuration actually matches what the code accepts. It's DRY principle applied to docs: write the schema once, generate human-readable docs and programmatic validation from the same source.

### 5. API Stability Levels

Explicit alpha → beta → v1 graduation with compatibility guarantees:
- `/v1alpha/*` - Breaking changes allowed
- `/v1beta/*` - Additive changes only
- `/v1/*` - Full backward compatibility

**Why this is interesting:** Most projects ship one API and then paint themselves into a corner when they need to make breaking changes (either break users or carry compatibility debt forever). OGX learned from Kubernetes: explicit API levels with documented stability contracts let you iterate on new APIs (`/v1alpha`) without committing to backward compatibility, graduate them when they're stable (`/v1beta`), and only lock into永久 compatibility when battle-tested (`/v1`). The pre-commit hook actually enforces this—breaking changes to `/v1` APIs are blocked at CI time. This gives the project room to experiment (alpha), collect feedback before freezing the contract (beta), and provide strong guarantees where it matters (v1), all within the same codebase. Users know exactly what they're signing up for based on the URL path.

### 6. Test Recording Hash-Based Matching

Integration tests match requests to recordings by SHA256 hash of request body. Deterministic and fast.

**Why this is interesting:** Most record/replay systems use sequential playback (first request → first recording, second request → second recording), which is brittle—reorder your test logic and all replays break. OGX's hash-based matching is content-addressable: the system hashes the request parameters (model, messages, temperature, etc.) and uses that as a lookup key. Same request → same hash → same cached response, regardless of order. This makes tests robust to refactoring. It also enables smart cache sharing: multiple tests making identical requests to the provider all get the same cached response, reducing the number of expensive API calls during recording. The deterministic ID override (using the hash to seed ID generation during replay) means resource IDs are reproducible across runs, so a test that creates a file and references it later works identically in record and replay modes.

### 7. Dependency-Ordered Provider Resolution

Providers declare dependencies (e.g., agents depends on inference). Resolver sorts topologically before instantiation.

**Why this is interesting:** When you have 20+ providers with interdependencies (the Responses provider needs an Inference provider, tool runtime needs access to registered tool groups, etc.), the initialization order matters. Get it wrong and you'll hit "provider not ready" errors at startup. Manual ordering is fragile and doesn't scale. OGX's resolver builds a dependency graph from provider specs, performs a topological sort, and initializes providers in an order that guarantees dependencies are ready before their consumers start. This handles complex dependency DAGs automatically—cycles are detected and reported as errors, and the same resolution logic works whether you're running one provider or fifty. It's the kind of "boring infrastructure" that lets you add new providers without worrying about initialization races or ordering bugs.

---

## Production Readiness Signals

This is **not a toy project**. Evidence of production-grade engineering:

✅ **Comprehensive testing** - 306 test files, record/replay system, CI workflows  
✅ **API stability contract** - Explicit versioning and upgrade guarantees  
✅ **Security** - CVE tracking, authorization checks, pre-commit safety hooks  
✅ **Multi-tenancy** - Routing tables, access control, quota middleware  
✅ **Observability** - OpenTelemetry integration, structured logging  
✅ **Storage flexibility** - SQLite for dev, PostgreSQL/Redis for production  
✅ **Documentation** - Every major directory has README, full Docusaurus site  
✅ **Release process** - Documented versioning, changelog, process  
✅ **Code quality** - Type hints, mypy, pre-commit hooks, conventional commits  
✅ **Performance** - Client reuse, streaming, bounded reads  

---

## Quick Start Commands

```bash
# Install
curl -LsSf https://github.com/ogx-ai/ogx/raw/main/scripts/install.sh | bash
# or
uv pip install ogx[starter]

# Start server (uses starter distribution with Ollama)
uv run ogx stack run starter

# Run unit tests
./scripts/unit-tests.sh

# Run integration tests (replay mode)
uv run ./scripts/integration-tests.sh \
  --stack-config server:ci-tests --setup gpt --suite responses

# Pre-commit checks
uv run pre-commit run --all-files

# Regenerate provider docs
uv run ./scripts/provider_codegen.py

# Regenerate OpenAPI specs
uv run ./scripts/run_openapi_generator.sh
```

---

## Areas for Deep Dive

If you want to explore further, these are the most interesting areas:

1. **Provider implementation patterns** - How OpenAIMixin works, adapter patterns
2. **Routing system internals** - Auto-routing, resource resolution
3. **Responses API orchestration** - How agentic workflows are implemented
4. **Test recording system** - Request hashing, deterministic ID generation
5. **Configuration resolution** - Environment variable substitution, conditionals
6. **Storage abstraction** - KVStore/SqlStore interface design
7. **Multi-SDK support** - How Anthropic and Google APIs coexist with OpenAI
8. **Distribution codegen** - Template system for different environments

---

## Glossary

**Monkey-patching** - Modifying code behavior at runtime by replacing methods or attributes on existing classes/modules. In OGX's test recorder, the `OpenAI` client class methods are replaced with wrappers that intercept calls, record them, and either forward to the real API (record mode) or return cached responses (replay mode). Called "monkey-patching" because you're dynamically changing how code behaves after it's already been defined.

**Topological sort** - An ordering of nodes in a directed graph such that for every edge from node A to node B, A comes before B in the ordering. OGX uses this to initialize providers in dependency order: if provider X depends on provider Y, the topological sort ensures Y is initialized first.

**Content-addressable storage** - A system where data is retrieved by its content (typically via a hash) rather than by location or name. OGX's test recordings use SHA256 hashes of request bodies as keys, so identical requests always map to the same cached response regardless of when/where they're made.

**Pluggable provider architecture** - A design where components (providers) can be swapped without changing the core system. OGX defines protocols (Python `Protocol` classes) for APIs like `Inference`, and any provider implementing that protocol can be used interchangeably.

**Server-side agentic loop** - Orchestration logic (model calls → tool execution → synthesis) that runs on the server rather than in client code. Contrast with client-side loops where the application must manually manage the iteration between model and tools.

**Distribution (in OGX context)** - A pre-configured bundle of providers and settings for a specific deployment scenario (like Kubernetes distributions: EKS, AKS, GKE). OGX distributions like "starter" or "nvidia" wire together different providers but expose the same API surface.

**12-factor app** - A methodology for building software-as-a-service apps. OGX follows principle III (store config in environment variables) via its `${env.VAR:=default}` substitution syntax in YAML configs.

**Pydantic** - Python library for data validation using type hints. OGX uses Pydantic models extensively for request/response validation and provider configuration, which provides runtime type checking and automatic serialization/deserialization.

**MCP (Model Context Protocol)** - An open standard for connecting AI models to external tools and data sources. OGX integrates MCP servers, allowing agents to discover and use tools automatically without custom integration code.

**RAG (Retrieval-Augmented Generation)** - A technique where a model's responses are grounded in retrieved documents from a knowledge base, reducing hallucination. OGX implements RAG server-side via the `file_search` tool.

---

## Summary

OGX is a **production-grade, provider-agnostic API server** that implements the OpenAI API (and Anthropic, Google) with a clean architecture:

- **Pluggable providers** - 19+ inference providers, multiple vector stores
- **Auto-routing** - Multiple providers serve same API, routed by resource ID
- **Multi-SDK** - OpenAI, Anthropic, Google SDKs work simultaneously
- **Flexible storage** - SQLite → PostgreSQL/Redis upgrade path
- **Sophisticated testing** - Record/replay system for fast, deterministic CI
- **Well-documented** - Module READMEs, full Docusaurus site, inline docs
- **Production-ready** - Security, observability, multi-tenancy, API stability

The codebase is clean, well-organized, and follows strong engineering practices. It's designed for real-world, multi-provider, multi-tenant deployments.
