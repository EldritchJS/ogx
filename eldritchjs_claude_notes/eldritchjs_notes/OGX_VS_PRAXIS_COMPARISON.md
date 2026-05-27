# OGX vs Praxis: Comparison

**Generated:** 2026-05-27  
**Purpose:** Understanding how OGX and Praxis differ and when to use each

---

## Executive Summary

**Praxis** and **OGX** solve different problems at different layers of the AI infrastructure stack:

- **Praxis** = High-performance reverse proxy/load balancer (like Nginx/Envoy for AI)
- **OGX** = Agentic orchestration server/API gateway (application server for AI)

They're **complementary, not competing**. You could use both together in a production deployment.

---

## Quick Comparison Table

| Aspect | Praxis | OGX |
|--------|--------|-----|
| **Language** | Rust | Python |
| **Primary Role** | Reverse proxy / Load balancer | Orchestration server / API gateway |
| **State** | Stateless (proxy) | Stateful (conversations, workflows) |
| **Scope** | Routes and transforms requests | Executes multi-step agentic workflows |
| **AI Features** | Routing, inspection, enrichment | Full agentic orchestration (RAG, tools) |
| **Focus** | Traffic management, security | Workload abstraction, governance |
| **Use Case** | "Route this request to the right backend" | "Execute this 5-step agentic workflow" |
| **Maturity (AI)** | Planned features (early stage) | Production-ready (Responses API, MCP) |
| **License** | MIT | MIT |

---

## What is Praxis?

### Overview

Praxis is "a high-performance and security-first proxy server and framework for AI and cloud-native workloads" written in Rust.

**GitHub:** https://github.com/praxis-proxy/praxis

### Key Features

#### **Traffic Management**
- Path, host, header routing with prefix-based matching
- Load balancing: round-robin, least-connections, consistent-hash, weighted endpoints
- Rate limiting (per-IP and global) with token bucket algorithm
- Circuit breaker per cluster with 503 responses
- Active/passive health checks
- Timeout enforcement (504 on SLA violations)

#### **Payload Processing**
- Zero-copy streaming by default
- **Body-based routing** - Extract `model` field from JSON, route to cluster
- Prompt enrichment for OpenAI-compatible chat completions
- Response compression (gzip, brotli, zstd)
- StreamBuffer pattern for inspecting request bodies before forwarding

#### **Security**
- Memory-safe: `#![deny(unsafe_code)]` in every crate
- Rustls-based TLS (no OpenSSL or C FFI)
- SSRF protection for health checks
- CORS filter with preflight handling
- IP ACL allow/deny lists
- CSRF protection

#### **Observability**
- Request ID generation and propagation
- Structured access logging
- Prometheus metrics at `/metrics`
- Admin health endpoints (`/ready`, `/healthy`)

#### **AI-Specific (Current)**
- Model-based routing (`model_to_header` filter)
- Credential injection per cluster
- StreamBuffer for request inspection

#### **AI-Specific (Planned)**
- Token counting
- Provider routing/failover
- Token-based rate limiting
- Cost attribution
- SSE inspection
- Semantic caching
- AI guardrails
- MCP proxying
- A2A (Agent-to-Agent) proxying

### Architecture

**Stateless proxy** with filter chain architecture:
```
Request → Listener → Filter Chain → Router → Load Balancer → Backend
```

Filter chains process requests/responses with conditional gates based on paths, methods, headers, status codes.

### Example Use Case

"Route requests with `model: gpt-4` to OpenAI cluster, `model: llama-3` to vLLM cluster, with rate limiting (100 req/min per IP) and circuit breaking"

---

## What is OGX?

### Overview

OGX is an open-source, multi-protocol AI orchestration server that provides server-side agentic workflows, provider abstraction, and centralized governance.

**GitHub:** https://github.com/ogx-ai/ogx

### Key Features

#### **Multi-Protocol API Gateway**
- Speaks OpenAI API natively
- Speaks Anthropic Messages API natively
- Speaks Google Interactions API natively
- All three SDKs work against same backend simultaneously

#### **Agentic Orchestration**
- **Responses API** - Server-side agentic loop (tool calling, multi-step reasoning)
- **file_search** tool - Built-in RAG with vector stores
- **MCP integration** - Connect MCP servers, auto-discover tools
- **Multi-step workflows** - Chain tool calls, synthesize results
- **Conversation state** - Persistent context across interactions

#### **Provider Abstraction**
- 20+ inference providers (vLLM, Ollama, OpenAI, Anthropic, Azure, Bedrock, etc.)
- Multiple vector stores (FAISS, pgvector, ChromaDB, Weaviate, etc.)
- **Swap providers without code changes** - Just update config

#### **Governance & Policy**
- **Per-team quotas** - Requests/day, tokens/request limits
- **Access control** - Which teams can use which models/tools
- **Cost tracking** - Attribute spend per team/project
- **Audit logging** - Who asked what, which tools were used

#### **Storage Flexibility**
- KVStore backends: SQLite, Redis, PostgreSQL, MongoDB
- SqlStore backends: SQLite, PostgreSQL
- Object storage for files (S3-compatible)

### Architecture

**Stateful orchestration engine** with pluggable providers:
```
Client SDK → OGX (API translation) → Orchestration Engine → Provider Router → Backend
                                    ↓
                            Conversation State
                            Vector Stores
                            Tool Runtime
```

### Example Use Case

"Execute a Responses API call that searches vector store for 'pricing strategy' docs, calls Jira MCP tool to get related issues, calls Confluence MCP tool to find related pages, then synthesizes all results into a grounded answer - and track this request against team-a's quota"

---

## Where They Fit in the Stack

```text
┌─────────────────────────────────────────────────┐
│  Application (Team's code)                      │
│  - Uses OpenAI/Anthropic/Google SDK             │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  Praxis (Optional - Edge Layer)                 │
│  - Load balancing                               │
│  - Rate limiting (per-IP, DDoS protection)      │
│  - Circuit breaking                             │
│  - Model-based routing                          │
│  - Security (CORS, IP ACL, CSRF)                │
│  - TLS termination                              │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  OGX (Orchestration & Abstraction Layer)        │
│  - API translation (multi-SDK support)          │
│  - Agentic orchestration (tool calling, RAG)    │
│  - Provider routing (vLLM, Ollama, cloud)       │
│  - Governance (quotas, access control)          │
│  - Conversation state management                │
│  - Cost tracking & attribution                  │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  Inference Backends                             │
│  - vLLM, Ollama, TGI, cloud providers           │
│  - Vector stores (pgvector, FAISS, etc.)        │
│  - Tool runtimes (MCP servers)                  │
└─────────────────────────────────────────────────┘
```

**Key insight:** They operate at different layers and can work together.

---

## Detailed Comparison

### Use Case: Simple Inference Request

**With Praxis:**
```text
1. Client sends request with model: "llama-3.3-70b"
2. Praxis extracts model from JSON body
3. Praxis routes to vLLM cluster based on model
4. Praxis forwards request to vLLM
5. vLLM returns response
6. Praxis returns to client
```

**What Praxis provides:** Fast routing, load balancing, rate limiting

**What Praxis doesn't do:** Multi-step workflows, tool calling, conversation state

---

**With OGX:**
```text
1. Client sends request with model: "llama-3.3-70b"
2. OGX checks team quota (has team-a used their daily limit?)
3. OGX translates API if needed (client used Anthropic SDK, backend is vLLM)
4. OGX routes to vLLM provider
5. vLLM returns response
6. OGX logs request for cost tracking
7. OGX returns to client
```

**What OGX provides:** Provider abstraction, quota enforcement, cost tracking, API translation

**What OGX doesn't do:** High-performance edge routing, per-IP rate limiting, circuit breaking

---

### Use Case: Agentic Workflow with Tools

**With Praxis:**
```text
Not designed for this. Praxis is a proxy, not an orchestrator.
Client would need to implement the orchestration loop themselves.
```

**With OGX:**
```text
1. Client sends Responses API request:
   - Input: "What's our Q1 revenue?"
   - Tools: [file_search, mcp:erp-system]

2. OGX orchestrates (server-side):
   a. Model generates search query: "Q1 revenue reports"
   b. OGX executes file_search on vector store
   c. Retrieves relevant docs
   d. Model decides to call erp-system MCP tool
   e. OGX calls MCP server with query
   f. Model synthesizes results from docs + ERP data
   
3. OGX returns final answer with citations

Client made ONE API call. Server handled entire workflow.
```

**What OGX provides:** Server-side orchestration, tool execution, RAG, synthesis

**What Praxis provides:** N/A (not designed for this)

---

### Use Case: Multi-Team Platform

**With Praxis:**
```text
Rate limiting:
  ✅ Per-IP rate limiting (prevent DDoS)
  ❌ Per-team quotas (can't identify teams from IP)
  
Routing:
  ✅ Route by model to different clusters
  ❌ Route by team to different providers
  
Security:
  ✅ IP ACL, CORS, CSRF protection
  ❌ Team-level access control (which models each team can use)
  
Cost tracking:
  ❌ No cost attribution per team
```

**With OGX:**
```text
Quotas:
  ✅ Per-team daily/hourly quotas
  ✅ Per-team token limits
  ✅ Per-team concurrent request limits
  
Routing:
  ✅ Route by team (team-a gets GPUs, team-b gets CPU)
  ✅ Route by model
  
Access control:
  ✅ Team-a can use gpt-4 and llama-3.3-70b
  ✅ Team-b can only use llama-3.2-3b
  ✅ Control which tools each team can access
  
Cost tracking:
  ✅ Cost per team/project
  ✅ Chargeback reports
```

---

## Performance Characteristics

### Praxis

**Strengths:**
- **Rust performance** - Zero-copy streaming, minimal overhead
- **Memory safety** - No unsafe code, prevents memory leaks
- **Low latency** - Designed for high-throughput proxying
- **Efficient** - Can handle thousands of concurrent connections

**Benchmarks:** (Planned in project, not yet published)

**Typical overhead:** ~1-5ms proxy latency

---

### OGX

**Strengths:**
- **Python ecosystem** - Easy integration with ML/AI libraries
- **Flexible** - Rapid development of new providers/features
- **Stateful** - Can maintain conversation context

**Trade-offs:**
- **Higher latency than Praxis** - Python overhead, orchestration logic
- **More resource intensive** - Stateful operations, database queries

**Typical overhead:** ~10-50ms for simple requests, more for orchestration

**When it matters:** Orchestration workflows are seconds-long anyway (model inference dominates), so 50ms OGX overhead is negligible

---

## Security Comparison

### Praxis Security Features

✅ **Memory safety** - Rust's ownership system prevents buffer overflows, use-after-free  
✅ **SSRF protection** - Blocks loopback, link-local, cloud metadata in health checks  
✅ **CORS** - Preflight handling, origin validation  
✅ **IP ACL** - Allow/deny lists  
✅ **CSRF protection** - `Sec-Fetch-Site` validation  
✅ **TLS** - Rustls (memory-safe, no C FFI)  
✅ **Supply chain** - `cargo audit` and `cargo deny` in CI  

**Focus:** Infrastructure security, prevent exploits at proxy layer

---

### OGX Security Features

✅ **Authorization** - Team-based access control  
✅ **Audit logging** - Who did what, when  
✅ **Data sovereignty** - Runs on-prem, data stays in your cluster  
✅ **Secrets management** - Integration with External Secrets Operator  
✅ **Network policies** - Works with OpenShift/Kubernetes NetworkPolicies  
✅ **Content filtering** - Guardrails for input/output (via providers)  

**Focus:** Application security, compliance, governance

---

## Maturity & Production Readiness

### Praxis

**Status:** Early stage, active development

**AI Features:**
- ✅ **Implemented:** Model-based routing, prompt enrichment
- 🔄 **Planned:** Token counting, semantic caching, MCP proxying, cost attribution

**General Features:**
- ✅ Load balancing, health checks, rate limiting, circuit breaking
- ✅ TLS, security hardening

**Community:** Smaller, newer project

**Use in production:** Likely too early for critical workloads (AI features still planned)

---

### OGX

**Status:** Production-ready (v1.0.3+)

**AI Features:**
- ✅ **Implemented:** Responses API, file_search, MCP integration, multi-step orchestration
- ✅ **Battle-tested:** Used in production deployments
- ✅ **Open Responses conformant** - Passes conformance test suite

**Providers:**
- ✅ 20+ inference providers
- ✅ Multiple vector stores
- ✅ Tool runtimes (MCP, custom)

**Community:** Larger, more established (originally Meta's Llama Stack, now community-driven)

**Use in production:** Ready for enterprise deployments

---

## When to Use Which

### Choose Praxis If:

✅ You need **high-performance edge routing**  
✅ You need **per-IP rate limiting** (DDoS protection)  
✅ You need **circuit breaking** across multiple backends  
✅ You want **Rust performance** and memory safety  
✅ Your use case is **simple routing** (not complex orchestration)  
✅ You need **low-latency proxy layer** (every millisecond counts)  

**Example:** "Route incoming requests to 5 different vLLM clusters based on model name, with rate limiting and circuit breaking"

---

### Choose OGX If:

✅ You need **server-side agentic orchestration**  
✅ You need **RAG** with vector stores (file_search)  
✅ You need **MCP tool integration**  
✅ You need **multi-SDK support** (OpenAI, Anthropic, Google)  
✅ You need **provider abstraction** (swap backends without code changes)  
✅ You need **governance** (quotas, cost tracking, access control)  
✅ You need **conversation state** management  

**Example:** "Build a multi-tenant AI platform where teams use different SDKs, access different models based on permissions, with RAG and tool calling orchestrated server-side"

---

### Use Both If:

✅ You're building an **enterprise AI platform**  
✅ You need **Praxis for edge security/performance** + **OGX for orchestration**  
✅ You want **layered defense** (Praxis handles traffic attacks, OGX handles application security)  

**Example Architecture:**
```text
Internet → Praxis (edge) → OGX (orchestration) → Inference backends

Praxis handles:
- DDoS protection (rate limiting by IP)
- Load balancing across OGX instances
- TLS termination
- Circuit breaking
- Fast routing decisions

OGX handles:
- Agentic workflows (RAG, tools, multi-step)
- Provider abstraction
- Team quotas and access control
- Cost tracking
- Conversation state
```

---

## Use Case Examples

### Scenario 1: Simple API Gateway

**Requirement:** Route requests to different backends based on model name

**Praxis approach:**
```yaml
# Extract model from JSON body, route to cluster
filters:
  - name: model_router
    type: model_to_header
    
routes:
  - path: /v1/chat/completions
    clusters:
      - name: vllm_cluster
        when: {header: x-model, value: "llama.*"}
      - name: openai_cluster
        when: {header: x-model, value: "gpt.*"}
```

**OGX approach:**
```yaml
# Register models with providers
registered_resources:
  models:
    - model_id: llama-3.3-70b
      provider_id: vllm
    - model_id: gpt-4
      provider_id: openai

# OGX automatically routes based on model_id
```

**Winner:** Either works. Praxis is faster (Rust), OGX is more flexible (easier to add providers).

---

### Scenario 2: RAG Application

**Requirement:** Search vector store for relevant docs, synthesize with LLM

**Praxis approach:**
```
Not designed for this. Client must:
1. Call vector store API to search
2. Call LLM with retrieved context
3. Handle orchestration in application code
```

**OGX approach:**
```python
# One API call, server handles orchestration
response = client.responses.create(
    model="llama-3.3-70b",
    input="What's our refund policy?",
    tools=[{"type": "file_search", "vector_store_ids": ["vs_123"]}]
)
# OGX searches vector store, synthesizes answer, returns with citations
```

**Winner:** OGX. RAG orchestration is its core feature. Praxis isn't designed for this.

---

### Scenario 3: Multi-Team Governance

**Requirement:** 
- Team A can use expensive models (gpt-4), limited to 1000 req/day
- Team B can use cheap models (llama-3.2-3b), limited to 10000 req/day
- Track costs per team

**Praxis approach:**
```
Limited support:
- Can do per-IP rate limiting (but teams != IPs)
- Cannot identify teams from requests
- No cost tracking
```

**OGX approach:**
```yaml
access_control:
  - principal: "team:team-a"
    allowed_models: ["gpt-4", "llama-3.3-70b"]
    quota: {requests_per_day: 1000}
  
  - principal: "team:team-b"
    allowed_models: ["llama-3.2-3b"]
    quota: {requests_per_day: 10000}

# OGX tracks costs per team automatically
```

**Winner:** OGX. Team-based governance is a core feature. Praxis lacks team awareness.

---

### Scenario 4: High-Throughput Inference

**Requirement:** Handle 10,000 req/sec with minimal latency

**Praxis approach:**
```
✅ Designed for this
- Rust performance
- Zero-copy streaming
- Minimal overhead (~1-5ms)
- Async I/O, efficient connection pooling
```

**OGX approach:**
```
⚠️ Not optimized for extreme throughput
- Python overhead
- Stateful operations (DB queries)
- Higher latency (~10-50ms)
- Can scale horizontally, but less efficient per instance
```

**Winner:** Praxis. High-throughput proxying is its strength.

---

## Can They Work Together?

**Yes!** They're complementary:

```text
Architecture: Praxis → OGX → Backends

┌─────────────────────────────────────────┐
│  Praxis (Edge Layer)                    │
│  - Handles external traffic             │
│  - Rate limiting (per-IP, DDoS)         │
│  - TLS termination                      │
│  - Load balancing across OGX instances  │
│  - Circuit breaking                     │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  OGX (Orchestration Layer)              │
│  - Agentic workflows                    │
│  - Provider abstraction                 │
│  - Team quotas & access control         │
│  - Cost tracking                        │
│  - Conversation state                   │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Inference Backends                     │
│  - vLLM, Ollama, cloud providers        │
└─────────────────────────────────────────┘
```

**Benefits of using both:**
1. **Praxis handles edge concerns** - DDoS protection, fast routing
2. **OGX handles application logic** - Orchestration, governance
3. **Separation of concerns** - Each tool does what it's best at
4. **Layered security** - Defense in depth

**Trade-off:** Added complexity (two services to operate)

---

## Development & Community

### Praxis

**Language:** Rust  
**License:** MIT  
**Repository:** https://github.com/praxis-proxy/praxis  
**Maintainer:** praxis-proxy organization  
**Community:** Early stage, smaller community  
**Contribution:** Open to contributions, Rust knowledge required  

**Documentation:**
- Quickstart guide
- Features overview
- Configuration examples
- Architecture docs

**Maturity:** Early stage (v0.x likely, though not specified)

---

### OGX

**Language:** Python (server), TypeScript (UI)  
**License:** MIT  
**Repository:** https://github.com/ogx-ai/ogx  
**Maintainer:** ogx-ai organization (formerly Meta's Llama Stack)  
**Community:** Larger, established community, weekly office hours  
**Contribution:** Active development, Python knowledge helpful  

**Documentation:**
- Full Docusaurus site
- API reference
- Provider guides
- Deployment examples
- Blog posts

**Maturity:** Production-ready (v1.0.3+)

---

## Technical Architecture Comparison

### Praxis Architecture

```text
┌─────────────────────────────────────────────┐
│  Listener (HTTP/TCP)                        │
│  - Binds to address:port                    │
│  - Handles TLS termination                  │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  Filter Chain (Extensible)                  │
│  - HttpFilter / TcpFilter traits            │
│  - Conditional gates (when/unless)          │
│  - Filters: router, load_balancer,          │
│    rate_limit, cors, etc.                   │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  Router                                     │
│  - Path/host/header matching                │
│  - Longest prefix wins                      │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  Load Balancer                              │
│  - Round-robin, least-conn, hash            │
│  - Health checking                          │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  Upstream Backend                           │
│  - HTTP/TCP connection                      │
└─────────────────────────────────────────────┘

Extension points:
- Custom filters (compile-time Rust code)
- Runtime key-value stores (pluggable backends)
```

---

### OGX Architecture

```text
┌─────────────────────────────────────────────┐
│  FastAPI Server                             │
│  - Multi-protocol endpoints                 │
│  - OpenAI, Anthropic, Google APIs           │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  Routing Layer                              │
│  - API translation (SDK format → internal)  │
│  - Request validation (Pydantic models)     │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  Orchestration Engine                       │
│  - Agentic loop execution                   │
│  - Tool calling (MCP, file_search)          │
│  - Multi-step workflows                     │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  Provider Registry & Router                 │
│  - Model → Provider mapping                 │
│  - Dynamic provider resolution              │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  Provider Implementations                   │
│  - Inference: vLLM, Ollama, cloud, etc.     │
│  - Vector IO: pgvector, FAISS, etc.         │
│  - Tool Runtime: MCP, custom tools          │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  Storage Layer                              │
│  - Conversations (PostgreSQL/SQLite)        │
│  - Metadata (Redis/PostgreSQL)              │
│  - Files (S3/local filesystem)              │
└─────────────────────────────────────────────┘

Extension points:
- Provider plugins (Python modules)
- Storage backends (pluggable)
- Custom APIs
```

---

## Feature Matrix

| Feature | Praxis | OGX |
|---------|--------|-----|
| **Protocol Support** |
| HTTP/1.1 | ✅ | ✅ |
| HTTP/2 | ✅ | ✅ |
| HTTP/3 | ✅ | ❌ |
| TCP | ✅ | ❌ |
| TLS/mTLS | ✅ | ✅ (via deployment) |
| **Routing** |
| Path-based | ✅ | ✅ |
| Header-based | ✅ | ✅ |
| Body-based (JSON model field) | ✅ | ✅ |
| Host-based | ✅ | ❌ |
| **Load Balancing** |
| Round-robin | ✅ | ✅ (via providers) |
| Least connections | ✅ | ❌ |
| Consistent hashing | ✅ | ❌ |
| Weighted | ✅ | ❌ |
| **Traffic Management** |
| Rate limiting | ✅ (per-IP, global) | ✅ (per-team, quotas) |
| Circuit breaking | ✅ | ❌ |
| Timeout enforcement | ✅ | ✅ |
| Retries | ✅ | ❌ |
| Health checks | ✅ (active/passive) | ❌ |
| **Security** |
| IP ACL | ✅ | ❌ |
| CORS | ✅ | ✅ (via deployment) |
| CSRF protection | ✅ | ❌ |
| SSRF protection | ✅ | ❌ |
| Memory safety | ✅ (Rust) | ⚠️ (Python) |
| **AI Features** |
| Model-based routing | ✅ | ✅ |
| Prompt enrichment | ✅ | ❌ |
| Token counting | 🔄 Planned | ❌ |
| Semantic caching | 🔄 Planned | ❌ |
| Agentic orchestration | ❌ | ✅ |
| RAG (file_search) | ❌ | ✅ |
| MCP integration | 🔄 Planned | ✅ |
| Multi-step workflows | ❌ | ✅ |
| Conversation state | ❌ | ✅ |
| **Observability** |
| Prometheus metrics | ✅ | ✅ |
| Structured logging | ✅ | ✅ |
| Request tracing | ⚠️ (planned?) | ✅ (OpenTelemetry) |
| Admin API | ✅ | ✅ |
| **Operational** |
| Hot reload config | ✅ | ❌ (restart required) |
| Graceful shutdown | ✅ | ✅ |
| Zero-downtime deploy | ✅ | ✅ (rolling update) |
| **Multi-Tenancy** |
| Per-IP isolation | ✅ | ❌ |
| Per-team isolation | ❌ | ✅ |
| Team quotas | ❌ | ✅ |
| Cost tracking | ❌ | ✅ |
| Access control | ⚠️ (IP-based) | ✅ (team-based) |
| **API Support** |
| OpenAI API | ✅ (proxy) | ✅ (native) |
| Anthropic API | ✅ (proxy) | ✅ (native) |
| Google API | ✅ (proxy) | ✅ (native) |
| Custom APIs | ✅ (via filters) | ✅ (via providers) |
| **Provider Abstraction** |
| Multiple backends | ✅ (via clusters) | ✅ (20+ providers) |
| Provider swapping | ⚠️ (config change) | ✅ (no code changes) |
| Multi-SDK support | ❌ | ✅ |

Legend:
- ✅ Implemented
- 🔄 Planned
- ⚠️ Partial support
- ❌ Not supported

---

## Analogies

### Praxis = Nginx/Envoy for AI

Just as Nginx/Envoy are high-performance proxies for web traffic, Praxis is a high-performance proxy optimized for AI workloads.

**Nginx/Envoy strengths:**
- Fast routing
- Load balancing
- Health checks
- TLS termination

**Praxis adds:**
- AI-aware routing (extract model from JSON)
- Prompt enrichment
- Planned: token counting, semantic caching

### OGX = Application Server for AI

Just as traditional app servers (Django, Rails, Express) handle business logic, OGX handles AI-specific orchestration logic.

**App server strengths:**
- Request handling
- Business logic
- Database integration
- Session management

**OGX adds:**
- Agentic workflows
- Tool orchestration
- Provider abstraction
- AI-specific governance

---

## Real-World Deployment Scenarios

### Scenario: Small Startup

**Team:** 5 engineers  
**Workload:** Simple chatbot, one model  
**Infrastructure:** Ollama on a single server  

**Recommendation:** Neither Praxis nor OGX needed initially

**Why:**
- Direct Ollama access is simpler
- No multi-team governance needed
- No complex orchestration needed
- Keep it simple: `client → Ollama`

**When to add OGX:** When you need RAG, tool calling, or multi-model support  
**When to add Praxis:** When you need rate limiting, multiple backends, or DDoS protection

---

### Scenario: Mid-Sized Company

**Team:** 20 engineers, 3 product teams  
**Workload:** Multiple AI features (RAG, agents, simple inference)  
**Infrastructure:** vLLM on GPUs, Ollama for dev, considering cloud burst  

**Recommendation:** OGX

**Why:**
- Multiple teams need governance
- Mix of simple inference and agentic workflows
- Need provider flexibility (vLLM → cloud)
- Cost tracking required

**Architecture:**
```text
Teams → OGX → vLLM (production) / Ollama (dev) / Cloud (burst)
```

**Praxis:** Optional, add if you need edge rate limiting or DDoS protection

---

### Scenario: Enterprise

**Team:** 100+ engineers, 20+ product teams  
**Workload:** Full range (simple inference, RAG, complex agents)  
**Infrastructure:** Multi-cluster OpenShift, hybrid cloud  

**Recommendation:** Both Praxis and OGX

**Why:**
- Need edge security (Praxis)
- Need orchestration & governance (OGX)
- Need layered defense
- Scale requires both performance (Praxis) and abstraction (OGX)

**Architecture:**
```text
Internet → Praxis (edge) → OGX (orchestration) → 
  → vLLM (on-prem GPUs)
  → Ollama (dev/small models)
  → Cloud providers (burst, specific models)
```

**Praxis responsibilities:**
- Rate limiting (per-IP, DDoS protection)
- Load balancing across OGX instances
- Circuit breaking
- TLS termination
- Fast routing

**OGX responsibilities:**
- Agentic workflows
- Team quotas & access control
- Provider abstraction
- Cost tracking
- Conversation state
- RAG orchestration

---

## Summary

### Praxis

**What it is:** High-performance Rust proxy for AI workloads  
**Strengths:** Speed, security, traffic management  
**Weaknesses:** No orchestration, AI features still planned  
**Best for:** Edge layer, high-throughput routing, DDoS protection  
**Maturity:** Early stage  

### OGX

**What it is:** Python orchestration server with provider abstraction  
**Strengths:** Agentic workflows, governance, multi-SDK support  
**Weaknesses:** Higher latency than Praxis, Python overhead  
**Best for:** Orchestration layer, multi-team platforms, RAG/tools  
**Maturity:** Production-ready  

### Together

**When to use both:** Enterprise deployments needing edge security + orchestration  
**Architecture:** Praxis (edge) → OGX (orchestration) → Backends  
**Trade-off:** More complex, but separates concerns cleanly  

### Neither

**When to skip both:** Small projects with simple requirements - direct backend access is simpler

---

## Further Reading

### Praxis
- GitHub: https://github.com/praxis-proxy/praxis
- Docs: https://github.com/praxis-proxy/praxis/tree/main/docs

### OGX
- GitHub: https://github.com/ogx-ai/ogx
- Docs: https://ogx-ai.github.io/docs
- Discord: https://discord.gg/ZAFjsrcw

### Related
- For OGX on OpenShift deployment: See `OGX_OPENSHIFT_GUIDE.md` in this directory
- For OGX architecture deep dive: See `CODEBASE_OVERVIEW.md` in this directory
- For OGX value proposition: See `PROBLEM_SPACE_OVERVIEW.md` in this directory

---

**Document Version:** 1.0  
**Last Updated:** 2026-05-27  
**Author:** eldritchjs
