# OGX on OpenShift: Complete Deployment Guide

**Generated:** 2026-05-26  
**Audience:** Platform engineers, DevOps teams, AI infrastructure architects  
**Purpose:** Comprehensive guide for deploying and operating OGX on Red Hat OpenShift

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [When OGX Makes Sense on OpenShift](#when-ogx-makes-sense-on-openshift)
4. [Deployment Patterns](#deployment-patterns)
5. [Multi-Tenancy & Governance](#multi-tenancy--governance)
6. [Security & Compliance](#security--compliance)
7. [Scaling & Performance](#scaling--performance)
8. [Integration with OpenShift Features](#integration-with-openshift-features)
9. [Operational Guide](#operational-guide)
10. [Troubleshooting](#troubleshooting)
11. [Migration Strategies](#migration-strategies)
12. [Best Practices](#best-practices)

---

## Executive Summary

### What is OGX?

OGX is an open-source, multi-protocol AI orchestration server that provides:
- **Unified API** across OpenAI, Anthropic, and Google SDKs
- **Provider abstraction** for 20+ inference backends (vLLM, Ollama, cloud providers)
- **Server-side orchestration** for agentic workflows (tool calling, RAG, multi-step reasoning)
- **Centralized governance** (access control, quotas, cost tracking)

### What is OpenShift?

Red Hat OpenShift is an enterprise Kubernetes distribution providing:
- **Container orchestration** with enterprise features
- **Multi-tenancy** via projects/namespaces
- **Developer platform** with integrated CI/CD (Tekton)
- **Security** hardening and compliance certifications
- **GPU scheduling** via Node Feature Discovery and GPU Operator

### How They Fit Together

```text
┌────────────────────────────────────────────────────────────┐
│  OpenShift Platform                                        │
│  ├─ Container orchestration (Kubernetes)                   │
│  ├─ Infrastructure multi-tenancy (projects/RBAC)           │
│  ├─ Service mesh (Istio/OpenShift Service Mesh)            │
│  ├─ GPU scheduling (NVIDIA GPU Operator)                   │
│  └─ CI/CD (OpenShift Pipelines/Tekton)                     │
└────────────────────────────────────────────────────────────┘
                           ↓
              OGX runs as a containerized service
                           ↓
┌────────────────────────────────────────────────────────────┐
│  OGX Layer                                                 │
│  ├─ AI API gateway (multi-SDK support)                     │
│  ├─ Orchestration engine (agentic loops, RAG)              │
│  ├─ AI workload governance (model access, quotas)          │
│  ├─ Provider routing (vLLM, Ollama, cloud)                 │
│  └─ Policy enforcement (cost control, compliance)          │
└────────────────────────────────────────────────────────────┘
                           ↓
              Routes to AI infrastructure
                           ↓
┌────────────────────────────────────────────────────────────┐
│  AI Services (all running on OpenShift)                    │
│  ├─ vLLM (GPU inference)                                   │
│  ├─ Ollama (CPU inference)                                 │
│  ├─ PostgreSQL + pgvector (vector storage)                 │
│  ├─ Redis (caching)                                        │
│  └─ Model storage (PVCs, object storage)                   │
└────────────────────────────────────────────────────────────┘
```

**The value proposition:**
- **OpenShift** provides the container platform and infrastructure governance
- **OGX** provides the AI workload abstraction and orchestration layer
- **Together** they enable enterprise AI deployments with proper multi-tenancy, security, and operational maturity

---

## Architecture Overview

### High-Level Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│  Application Tier (Multiple Teams)                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Team A Apps  │  │ Team B Apps  │  │ Team C Apps  │          │
│  │ Namespace:   │  │ Namespace:   │  │ Namespace:   │          │
│  │ team-a       │  │ team-b       │  │ team-c       │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                  │
│         └──────────────────┴──────────────────┘                 │
│                            │                                     │
│         All use OpenAI/Anthropic/Google SDKs                    │
│                            │                                     │
└────────────────────────────┼─────────────────────────────────────┘
                             ↓
                  OGX Service (ClusterIP/Route)
                  http://ogx-api.ai-platform.svc
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  AI Platform Tier                                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ OGX Server Deployment                                     │  │
│  │ Namespace: ai-platform                                    │  │
│  │ ┌─────────┐  ┌─────────┐  ┌─────────┐                    │  │
│  │ │ OGX Pod │  │ OGX Pod │  │ OGX Pod │  (3 replicas)      │  │
│  │ └─────────┘  └─────────┘  └─────────┘                    │  │
│  │                                                            │  │
│  │ Features:                                                  │  │
│  │ - Multi-SDK API translation                               │  │
│  │ - Orchestration engine (Responses API)                    │  │
│  │ - Model routing (vLLM, Ollama, cloud)                     │  │
│  │ - Access control & quotas                                 │  │
│  │ - Observability (OpenTelemetry)                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                             │                                    │
│              Routes to backend services                          │
│                             │                                    │
└─────────────────────────────┼────────────────────────────────────┘
                              ↓
            ┌─────────────────┴─────────────────┐
            │                                   │
            ↓                                   ↓
┌─────────────────────────┐      ┌─────────────────────────┐
│ Inference Tier          │      │ Storage Tier            │
│ Namespace: ai-inference │      │ Namespace: ai-storage   │
│                         │      │                         │
│ ┌─────────────────────┐ │      │ ┌─────────────────────┐ │
│ │ vLLM Deployment     │ │      │ │ PostgreSQL +        │ │
│ │ - GPU: 4x A100      │ │      │ │ pgvector            │ │
│ │ - Model: Llama 70B  │ │      │ │                     │ │
│ │ - Replicas: 2       │ │      │ │ - Vector storage    │ │
│ └─────────────────────┘ │      │ │ - Metadata DB       │ │
│                         │      │ └─────────────────────┘ │
│ ┌─────────────────────┐ │      │                         │
│ │ Ollama Deployment   │ │      │ ┌─────────────────────┐ │
│ │ - CPU only          │ │      │ │ Redis               │ │
│ │ - Model: Llama 3B   │ │      │ │                     │ │
│ │ - Replicas: 3       │ │      │ │ - OGX cache         │ │
│ └─────────────────────┘ │      │ │ - Session storage   │ │
│                         │      │ └─────────────────────┘ │
│ ┌─────────────────────┐ │      │                         │
│ │ Model Storage       │ │      │ ┌─────────────────────┐ │
│ │ - PVC: 500Gi        │ │      │ │ Object Storage      │ │
│ │ - Model weights     │ │      │ │ (S3/Minio)          │ │
│ │ - Shared across pods│ │      │ │                     │ │
│ └─────────────────────┘ │      │ │ - File uploads      │ │
│                         │      │ │ - Vector store data │ │
└─────────────────────────┘      │ └─────────────────────┘ │
                                 └─────────────────────────┘
```

### Network Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│  Ingress Layer                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ OpenShift Router / Ingress Controller                    │  │
│  │ - External access via Routes                              │  │
│  │ - TLS termination                                         │  │
│  │ - Load balancing                                          │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                    Route: ogx-api.apps.cluster.example.com
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Service Mesh (Optional)                                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ OpenShift Service Mesh (Istio)                            │  │
│  │ - mTLS between services                                   │  │
│  │ - Traffic management                                      │  │
│  │ - Observability (distributed tracing)                     │  │
│  │ - Circuit breaking                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Internal Services (ClusterIP)                                  │
│                                                                  │
│  ogx-api.ai-platform.svc.cluster.local:8321                     │
│        ↓                                                         │
│        ├─→ vllm.ai-inference.svc.cluster.local:8000             │
│        ├─→ ollama.ai-inference.svc.cluster.local:11434          │
│        ├─→ postgresql.ai-storage.svc.cluster.local:5432         │
│        └─→ redis.ai-storage.svc.cluster.local:6379              │
│                                                                  │
│  Network Policies:                                               │
│  - Team namespaces → Can access ogx-api only                    │
│  - ai-platform → Can access ai-inference and ai-storage         │
│  - ai-inference → Isolated (no outbound except dns)             │
└─────────────────────────────────────────────────────────────────┘
```

### Storage Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│  Storage Classes & Provisioning                                 │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────┐  ┌──────────────────────┐
│ Model Storage (RWX)  │  │ Database (RWO)       │
│                      │  │                      │
│ StorageClass:        │  │ StorageClass:        │
│   nfs-storage        │  │   fast-ssd           │
│                      │  │                      │
│ Size: 500Gi          │  │ Size: 100Gi          │
│ AccessMode: RWX      │  │ AccessMode: RWO      │
│                      │  │                      │
│ Used by:             │  │ Used by:             │
│ - vLLM pods          │  │ - PostgreSQL         │
│ - Ollama pods        │  │ - Redis              │
│ - Model downloads    │  │                      │
└──────────────────────┘  └──────────────────────┘

┌──────────────────────┐  ┌──────────────────────┐
│ Object Storage (S3)  │  │ Ephemeral (EmptyDir) │
│                      │  │                      │
│ Provider:            │  │ Used for:            │
│   Minio/ODF          │  │ - Temporary files    │
│                      │  │ - Cache              │
│ Size: 1Ti            │  │ - /tmp               │
│                      │  │                      │
│ Used by:             │  │ Pod-local only       │
│ - File uploads       │  │                      │
│ - Vector store data  │  │                      │
│ - Backups            │  │                      │
└──────────────────────┘  └──────────────────────┘

Storage Strategy:
- Model weights: ReadWriteMany (RWX) NFS/GlusterFS for sharing across pods
- Databases: ReadWriteOnce (RWO) block storage for performance
- Files/vectors: S3-compatible object storage (Minio/OpenShift Data Foundation)
- Ephemeral: emptyDir for temporary data
```

---

## When OGX Makes Sense on OpenShift

### Decision Framework

**OGX adds value when you answer "yes" to 2+ of these:**

#### Complexity Indicators
- [ ] **Multiple inference backends** - Running vLLM + Ollama + TGI (or planning to)
- [ ] **Heterogeneous GPU infrastructure** - Mix of A100, V100, CPU-only nodes
- [ ] **Complex agentic workflows** - Need tool calling, RAG, multi-step reasoning
- [ ] **Multiple model versions** - Development, staging, production models
- [ ] **Hybrid deployment** - Some workloads on-prem, some in cloud

#### Organizational Indicators
- [ ] **Multi-team platform** - 3+ teams building AI applications
- [ ] **Centralized governance needed** - Platform team manages AI infrastructure
- [ ] **Cost accountability** - Need to track spending per team/project
- [ ] **Compliance requirements** - HIPAA, SOC2, FedRAMP, etc.
- [ ] **Standardization goals** - Want consistent API across all teams

#### Operational Indicators
- [ ] **Operational maturity** - Already running containerized services at scale
- [ ] **Platform engineering team** - Have capacity to operate additional services
- [ ] **Multi-SDK requirements** - Teams prefer different SDKs (OpenAI vs Anthropic vs Google)
- [ ] **Provider flexibility** - Want to swap backends without application changes
- [ ] **Future-proofing** - May migrate to cloud or change providers later

### Use Case Mapping

| Use Case | OpenShift Alone | OpenShift + OGX | Winner |
|----------|-----------------|-----------------|--------|
| **Single team, one model, simple inference** | Direct vLLM access, simple deployment | Additional complexity for little gain | OpenShift alone |
| **Multi-team, multiple models, governance needed** | Each team deploys own stack, duplicated effort | Centralized platform, shared governance | OpenShift + OGX |
| **Development environment only** | Docker Compose might suffice | Overkill for dev-only | Neither (use simpler tools) |
| **Production multi-tenant AI platform** | Complex custom routing, no standard API | Unified API, built-in governance | OpenShift + OGX |
| **Agentic applications with RAG** | Each app implements orchestration | Server-side orchestration, built-in RAG | OpenShift + OGX |
| **Air-gapped deployment** | Both work, no internet needed | Both work, OGX adds abstraction | OpenShift + OGX (if complex) |
| **Hybrid cloud/on-prem** | Manual routing per environment | Unified API, config-driven routing | OpenShift + OGX |
| **Cost optimization required** | Manual tracking per namespace | Built-in quotas, cost attribution | OpenShift + OGX |

### Comparison: Direct Provider Access vs. OGX

#### Scenario: Team wants to use vLLM for inference

**Option A: Direct vLLM Access (No OGX)**

```python
# Team's application code
from openai import OpenAI

client = OpenAI(
    base_url="http://vllm.ai-inference.svc.cluster.local/v1",
    api_key="fake"
)

response = client.chat.completions.create(
    model="llama-3.3-70b",
    messages=[{"role": "user", "content": "Hello"}]
)
```

**Pros:**
- ✅ Simple, direct connection
- ✅ One fewer service to operate
- ✅ Lower latency (no proxy hop)
- ✅ Easier to debug (fewer moving parts)

**Cons:**
- ❌ Hardcoded to vLLM (migration to TGI requires code changes)
- ❌ No quotas (team can use unlimited resources)
- ❌ No access control (all teams can access all models)
- ❌ No cost tracking (can't attribute spend)
- ❌ No orchestration (team must implement agentic loops)
- ❌ No multi-SDK (locked into OpenAI SDK pattern)

**Option B: Via OGX**

```python
# Team's application code (unchanged)
from openai import OpenAI

client = OpenAI(
    base_url="http://ogx-api.ai-platform.svc.cluster.local/v1",
    api_key="team-a-key"  # OGX tracks this
)

response = client.chat.completions.create(
    model="llama-3.3-70b",  # OGX routes to vLLM
    messages=[{"role": "user", "content": "Hello"}]
)
```

**Pros:**
- ✅ Platform can swap vLLM → TGI without team changes
- ✅ Quotas enforced (team-a limited to 10k requests/day)
- ✅ Access control (team-a can only use approved models)
- ✅ Cost tracking (all requests attributed to team-a)
- ✅ Orchestration available (Responses API with RAG, tools)
- ✅ Multi-SDK support (team can use Anthropic SDK if preferred)

**Cons:**
- ❌ Additional service to operate (OGX deployment)
- ❌ Slightly higher latency (~10-20ms proxy overhead)
- ❌ More complex debugging (OGX logs + vLLM logs)
- ❌ Operational overhead (monitoring, scaling OGX)

**When to choose Option A:**
- Single team, single model, simple use case
- No governance requirements
- Team fine with potential future migration work

**When to choose Option B:**
- Multi-team platform with governance needs
- Complex agentic workflows
- Want provider flexibility
- Cost/quota tracking required

---

## Deployment Patterns

### Pattern 1: Basic Single-Cluster Deployment

**Architecture:** All components in one OpenShift cluster

```text
OpenShift Cluster
├── Namespace: ai-platform
│   ├── OGX Deployment (3 replicas)
│   ├── OGX Service (ClusterIP)
│   └── OGX Route (external access)
├── Namespace: ai-inference
│   ├── vLLM Deployment (GPU)
│   └── Ollama Deployment (CPU)
├── Namespace: ai-storage
│   ├── PostgreSQL StatefulSet
│   ├── Redis Deployment
│   └── Minio (S3-compatible)
└── Namespace: team-a, team-b, team-c
    └── Application Deployments
```

**Deployment:**

```yaml
# 1. Create namespaces
---
apiVersion: v1
kind: Namespace
metadata:
  name: ai-platform
---
apiVersion: v1
kind: Namespace
metadata:
  name: ai-inference
---
apiVersion: v1
kind: Namespace
metadata:
  name: ai-storage

# 2. Deploy OGX
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ogx-server
  namespace: ai-platform
  labels:
    app: ogx-server
    version: v1.0.3
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ogx-server
  template:
    metadata:
      labels:
        app: ogx-server
        version: v1.0.3
    spec:
      serviceAccountName: ogx-server
      containers:
      - name: ogx
        image: docker.io/ogx/distribution-starter:1.0.3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8321
          name: http
          protocol: TCP
        env:
        # Inference provider URLs (internal services)
        - name: VLLM_URL
          value: "http://vllm.ai-inference.svc.cluster.local:8000/v1"
        - name: OLLAMA_URL
          value: "http://ollama.ai-inference.svc.cluster.local:11434"
        
        # Storage configuration
        - name: POSTGRES_HOST
          value: "postgresql.ai-storage.svc.cluster.local"
        - name: POSTGRES_DB
          value: "ogx"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: password
        
        - name: REDIS_URL
          value: "redis://redis.ai-storage.svc.cluster.local:6379"
        
        # OGX configuration
        - name: SQLITE_STORE_DIR
          value: "/data/ogx"
        
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8321
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8321
          initialDelaySeconds: 5
          periodSeconds: 5
        
        volumeMounts:
        - name: ogx-config
          mountPath: /etc/ogx/config.yaml
          subPath: config.yaml
        - name: data
          mountPath: /data
      
      volumes:
      - name: ogx-config
        configMap:
          name: ogx-config
      - name: data
        emptyDir: {}

---
# 3. OGX Service
apiVersion: v1
kind: Service
metadata:
  name: ogx-api
  namespace: ai-platform
  labels:
    app: ogx-server
spec:
  type: ClusterIP
  ports:
  - port: 8321
    targetPort: 8321
    protocol: TCP
    name: http
  selector:
    app: ogx-server

---
# 4. OGX Route (external access)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ogx-api
  namespace: ai-platform
spec:
  host: ogx-api.apps.cluster.example.com
  to:
    kind: Service
    name: ogx-api
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect

---
# 5. OGX ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: ogx-config
  namespace: ai-platform
data:
  config.yaml: |
    version: 2
    distro_name: openshift-starter
    
    apis:
      - inference
      - responses
      - vector_io
      - files
      - tool_runtime
    
    providers:
      inference:
        - provider_id: vllm
          provider_type: remote::vllm
          config:
            base_url: ${env.VLLM_URL}
            max_tokens: 4096
        
        - provider_id: ollama
          provider_type: remote::ollama
          config:
            base_url: ${env.OLLAMA_URL}
      
      vector_io:
        - provider_id: pgvector
          provider_type: remote::pgvector
          config:
            host: ${env.POSTGRES_HOST}
            port: 5432
            db: ${env.POSTGRES_DB}
            user: ${env.POSTGRES_USER}
            password: ${env.POSTGRES_PASSWORD}
      
      files:
        - provider_id: builtin-files
          provider_type: inline::localfs
          config:
            storage_dir: /data/ogx/files
      
      responses:
        - provider_id: builtin
          provider_type: inline::builtin
      
      tool_runtime:
        - provider_id: file-search
          provider_type: inline::file-search
    
    storage:
      backends:
        kv_default:
          type: kv_redis
          url: ${env.REDIS_URL}
        
        sql_default:
          type: sql_postgres
          host: ${env.POSTGRES_HOST}
          port: 5432
          db: ${env.POSTGRES_DB}
          user: ${env.POSTGRES_USER}
          password: ${env.POSTGRES_PASSWORD}
      
      stores:
        metadata:
          namespace: registry
          backend: kv_default
        
        inference:
          table_name: inference_store
          backend: sql_default
        
        conversations:
          table_name: conversations
          backend: sql_default
    
    server:
      port: 8321
    
    registered_resources:
      models:
        - model_id: llama-3.3-70b
          provider_id: vllm
          model_type: llm
        
        - model_id: llama-3.2-3b
          provider_id: ollama
          model_type: llm

---
# 6. ServiceAccount for OGX
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ogx-server
  namespace: ai-platform
```

### Pattern 2: Multi-Cluster with Centralized OGX

**Architecture:** OGX in management cluster, inference in workload clusters

```text
Management Cluster (OGX)          Workload Cluster 1 (GPU)
┌─────────────────────────┐       ┌─────────────────────────┐
│ Namespace: ai-platform  │       │ Namespace: ai-inference │
│ ├── OGX Deployment      │◄──────┤ vLLM (A100 GPUs)        │
│ ├── PostgreSQL          │       │ - Llama 70B             │
│ └── Redis               │       │ - Exposed via Route     │
└─────────────────────────┘       └─────────────────────────┘
                                  
                                  Workload Cluster 2 (CPU)
                                  ┌─────────────────────────┐
                                  │ Namespace: ai-inference │
                                  │ Ollama (CPU only)       │
                                  │ - Llama 3B              │
                                  │ - Exposed via Route     │
                                  └─────────────────────────┘

Communication:
- OGX → vLLM: https://vllm.cluster1.example.com
- OGX → Ollama: https://ollama.cluster2.example.com
- Cross-cluster TLS with mutual authentication
```

**When to use:**
- GPU clusters separate from general workload clusters
- Multi-datacenter deployment
- Cost optimization (GPU cluster different from management)

### Pattern 3: Air-Gapped Deployment

**Requirements:**
- No internet access
- All images in internal registry
- Model weights pre-loaded
- Certificates pre-deployed

**Preparation:**

```bash
# 1. Mirror images to internal registry
oc image mirror \
  docker.io/ogx/distribution-starter:1.0.3 \
  registry.internal.example.com/ogx/distribution-starter:1.0.3

oc image mirror \
  docker.io/vllm/vllm-openai:latest \
  registry.internal.example.com/ai/vllm-openai:latest

# 2. Download model weights (on internet-connected machine)
huggingface-cli download meta-llama/Llama-3.3-70B-Instruct

# 3. Transfer to air-gapped environment
tar -czf models.tar.gz ~/.cache/huggingface/
# Transfer via approved method (sneakernet, etc.)

# 4. Upload to internal registry or PVC
oc create -f model-pvc.yaml
oc rsync ~/.cache/huggingface/ model-pod:/models/
```

**Deployment adjustments:**

```yaml
# Use internal registry
spec:
  containers:
  - name: ogx
    image: registry.internal.example.com/ogx/distribution-starter:1.0.3
    imagePullPolicy: IfNotPresent  # Don't try to pull

# No external provider configs
providers:
  inference:
    - provider_id: vllm
      config:
        base_url: "http://vllm.ai-inference.svc.cluster.local:8000"
        # NO external API keys or URLs
```

### Pattern 4: High Availability with Service Mesh

**Components:**
- OpenShift Service Mesh (Istio)
- OGX with multiple replicas
- Circuit breaking and retries
- mTLS between services

```yaml
---
# ServiceMeshMemberRoll - add namespaces to mesh
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: istio-system
spec:
  members:
    - ai-platform
    - ai-inference
    - ai-storage

---
# DestinationRule for OGX (circuit breaking)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ogx-api
  namespace: ai-platform
spec:
  host: ogx-api.ai-platform.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 3
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50

---
# VirtualService for traffic splitting (A/B testing)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ogx-api
  namespace: ai-platform
spec:
  hosts:
  - ogx-api.ai-platform.svc.cluster.local
  http:
  - match:
    - headers:
        x-ogx-version:
          exact: "v2"
    route:
    - destination:
        host: ogx-api
        subset: v2
  - route:
    - destination:
        host: ogx-api
        subset: v1
      weight: 90
    - destination:
        host: ogx-api
        subset: v2
      weight: 10  # Canary 10% to new version
```

---

## Multi-Tenancy & Governance

### OpenShift Multi-Tenancy (Infrastructure Level)

**What OpenShift provides:**

```yaml
# 1. Project/Namespace isolation
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-a

---
# 2. Resource Quotas (CPU, Memory, GPU)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "32Gi"
    limits.cpu: "20"
    limits.memory: "64Gi"
    requests.nvidia.com/gpu: "2"  # Limit GPU access

---
# 3. RBAC (who can do what)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-developers
  namespace: team-a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- kind: Group
  name: team-a-developers
  apiGroup: rbac.authorization.k8s.io

---
# 4. Network Policies (network isolation)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ogx-only
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: openshift-dns
    ports:
    - protocol: UDP
      port: 53
  # Allow OGX API only
  - to:
    - namespaceSelector:
        matchLabels:
          name: ai-platform
    ports:
    - protocol: TCP
      port: 8321
```

### OGX Multi-Tenancy (AI Workload Level)

**What OGX adds on top:**

```yaml
# OGX config.yaml - AI-specific governance
access_control:
  # Team A: Data Science - full access
  - principal: "team:team-a"
    allowed_models:
      - "llama-3.3-70b"
      - "llama-3.2-3b"
      - "gpt-4"  # If cloud provider configured
    allowed_tools:
      - "file_search"
      - "mcp:jira"
      - "mcp:confluence"
    quota:
      requests_per_day: 10000
      requests_per_hour: 1000
      max_tokens_per_request: 8000
      max_concurrent_requests: 50
  
  # Team B: Customer Support - restricted access
  - principal: "team:team-b"
    allowed_models:
      - "llama-3.2-3b"  # Cheaper model only
    allowed_tools:
      - "file_search"  # RAG only, no external tools
    quota:
      requests_per_day: 50000
      requests_per_hour: 5000
      max_tokens_per_request: 2000
      max_concurrent_requests: 100
  
  # Team C: Experimentation - time-limited access
  - principal: "team:team-c"
    allowed_models:
      - "llama-3.3-70b"
    allowed_tools:
      - "file_search"
    quota:
      requests_per_day: 1000
      requests_per_hour: 100
      max_tokens_per_request: 4000
    valid_until: "2026-12-31T23:59:59Z"  # Expires after eval period
```

### Implementing Team-Based Authentication

**Option 1: API Keys (Simple)**

```yaml
# Secret per team
---
apiVersion: v1
kind: Secret
metadata:
  name: team-a-ogx-key
  namespace: team-a
type: Opaque
stringData:
  api-key: "team-a-production-key-abc123"

---
# Application uses secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chatbot
  namespace: team-a
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: OGX_API_KEY
          valueFrom:
            secretKeyRef:
              name: team-a-ogx-key
              key: api-key
        - name: OGX_BASE_URL
          value: "http://ogx-api.ai-platform.svc.cluster.local:8321/v1"
```

**Option 2: Service Account Tokens (OpenShift-native)**

```yaml
# OGX authenticates via ServiceAccount token
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: chatbot-sa
  namespace: team-a
  annotations:
    ogx.ai/team: "team-a"  # OGX reads this annotation

---
# Application pod uses ServiceAccount
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chatbot
  namespace: team-a
spec:
  template:
    spec:
      serviceAccountName: chatbot-sa
      containers:
      - name: app
        env:
        - name: OGX_BASE_URL
          value: "http://ogx-api.ai-platform.svc.cluster.local:8321/v1"
        volumeMounts:
        - name: sa-token
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      volumes:
      - name: sa-token
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600

# Application reads token and sends to OGX
# OGX validates token with Kubernetes API
# OGX extracts team from annotation
```

**Option 3: OAuth/OIDC (Enterprise)**

```yaml
# Integrate with corporate IdP
# OGX configured to validate JWT tokens from Keycloak/AAD/Okta

# Application obtains JWT
# Sends as Bearer token to OGX
# OGX validates signature, extracts claims (team, user)
# Enforces policy based on claims
```

### Cost Tracking & Chargeback

**OGX tracks usage per team:**

```sql
-- OGX inference_store table
SELECT
  principal,  -- team:team-a
  COUNT(*) as requests,
  SUM(input_tokens) as input_tokens,
  SUM(output_tokens) as output_tokens,
  SUM(total_cost) as total_cost_usd
FROM inference_store
WHERE timestamp >= '2026-05-01'
  AND timestamp < '2026-06-01'
GROUP BY principal;

-- Results:
-- team:team-a | 8543 | 1234567 | 987654 | $145.32
-- team:team-b | 45621 | 987654 | 654321 | $89.12
-- team:team-c | 432 | 12345 | 6789 | $2.15
```

**Chargeback report generation:**

```bash
# Monthly cost report
oc exec -n ai-platform ogx-server-0 -- \
  ogx admin cost-report \
    --start 2026-05-01 \
    --end 2026-06-01 \
    --format csv \
    > team_costs_may2026.csv

# Per-model breakdown
oc exec -n ai-platform ogx-server-0 -- \
  ogx admin cost-report \
    --start 2026-05-01 \
    --end 2026-06-01 \
    --group-by model,team \
    --format json \
    > detailed_costs_may2026.json
```

---

## Security & Compliance

### Network Security

**NetworkPolicy: Restrict egress from team namespaces**

```yaml
---
# Team-A can only talk to OGX, nothing else
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-dns
    ports:
    - protocol: UDP
      port: 53
  # OGX API only
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ai-platform
    - podSelector:
        matchLabels:
          app: ogx-server
    ports:
    - protocol: TCP
      port: 8321

---
# AI-platform can talk to inference and storage, but not teams
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ogx-backend-access
  namespace: ai-platform
spec:
  podSelector:
    matchLabels:
      app: ogx-server
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-dns
    ports:
    - protocol: UDP
      port: 53
  # Inference backends
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ai-inference
    ports:
    - protocol: TCP
      port: 8000  # vLLM
    - protocol: TCP
      port: 11434  # Ollama
  # Storage backends
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ai-storage
    ports:
    - protocol: TCP
      port: 5432  # PostgreSQL
    - protocol: TCP
      port: 6379  # Redis
```

### TLS/mTLS Configuration

**Service Mesh mTLS (automatic)**

```yaml
---
# PeerAuthentication - enforce mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ai-platform
spec:
  mtls:
    mode: STRICT  # Require mTLS for all traffic

---
# No code changes needed - Istio handles TLS automatically
# All communication encrypted:
# - Team apps → OGX
# - OGX → vLLM
# - OGX → PostgreSQL
```

**Manual TLS (without Service Mesh)**

```yaml
---
# Certificate for OGX
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ogx-api-tls
  namespace: ai-platform
spec:
  secretName: ogx-api-tls
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
  dnsNames:
  - ogx-api.ai-platform.svc.cluster.local
  - ogx-api.apps.cluster.example.com

---
# OGX Route with TLS
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ogx-api
  namespace: ai-platform
spec:
  host: ogx-api.apps.cluster.example.com
  to:
    kind: Service
    name: ogx-api
  port:
    targetPort: https
  tls:
    termination: reencrypt
    destinationCACertificate: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
```

### Secrets Management

**Option 1: OpenShift Secrets (Basic)**

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-credentials
  namespace: ai-platform
type: Opaque
stringData:
  username: ogx_user
  password: "super-secret-password"
  host: postgresql.ai-storage.svc.cluster.local
  database: ogx
```

**Option 2: External Secrets Operator (Enterprise)**

```yaml
---
# SecretStore pointing to Vault
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: ai-platform
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "ai-platform"

---
# ExternalSecret syncs from Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: postgresql-credentials
  namespace: ai-platform
spec:
  refreshInterval: 15s
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: postgresql-credentials
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: ai-platform/postgresql
      property: username
  - secretKey: password
    remoteRef:
      key: ai-platform/postgresql
      property: password
```

### Compliance: Audit Logging

**OGX audit log configuration:**

```yaml
# OGX config.yaml
server:
  audit:
    enabled: true
    backend: sql_default
    log_level: info
    include_request_body: false  # Don't log user prompts (PII)
    include_response_body: false  # Don't log model responses (PII)
    fields:
      - timestamp
      - principal  # team:team-a
      - model
      - provider
      - duration_ms
      - status_code
      - input_tokens
      - output_tokens
```

**Query audit logs:**

```sql
-- All requests from team-a in last 24h
SELECT *
FROM audit_log
WHERE principal = 'team:team-a'
  AND timestamp > NOW() - INTERVAL '24 hours'
ORDER BY timestamp DESC;

-- Failed requests (potential security issues)
SELECT *
FROM audit_log
WHERE status_code >= 400
  AND timestamp > NOW() - INTERVAL '7 days'
ORDER BY timestamp DESC;

-- High-cost requests (potential abuse)
SELECT *
FROM audit_log
WHERE total_cost_usd > 1.00
  AND timestamp > NOW() - INTERVAL '24 hours'
ORDER BY total_cost_usd DESC;
```

### Data Residency & Sovereignty

**All data stays in OpenShift cluster:**

```text
┌────────────────────────────────────────────────┐
│  Data Flow (On-Prem Only)                      │
├────────────────────────────────────────────────┤
│                                                │
│  1. User prompt → Team app (namespace: team-a) │
│  2. Team app → OGX (namespace: ai-platform)    │
│  3. OGX → vLLM (namespace: ai-inference)       │
│  4. vLLM processes locally (no external calls) │
│  5. Response → OGX → Team app                  │
│  6. Logs → PostgreSQL (namespace: ai-storage)  │
│                                                │
│  ✅ All data stays within cluster boundary     │
│  ✅ No external API calls (if using on-prem)   │
│  ✅ Model weights on local PVC                 │
│  ✅ Vector embeddings in local pgvector        │
│                                                │
└────────────────────────────────────────────────┘
```

**Compliance verification:**

```yaml
# NetworkPolicy: Deny all egress except internal
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external-egress
  namespace: ai-inference
spec:
  podSelector:
    matchLabels:
      app: vllm
  policyTypes:
  - Egress
  egress:
  # Only internal cluster traffic allowed
  - to:
    - podSelector: {}
  # Explicitly deny internet
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8      # Internal cluster network
        - 172.16.0.0/12   # Internal cluster network
        - 192.168.0.0/16  # Internal cluster network
```

---

## Scaling & Performance

### Horizontal Pod Autoscaling (HPA)

**OGX autoscaling based on request rate:**

```yaml
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ogx-server-hpa
  namespace: ai-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ogx-server
  minReplicas: 3
  maxReplicas: 10
  metrics:
  # Scale on CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Scale on memory utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Scale on request rate (custom metric from Prometheus)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
      policies:
      - type: Percent
        value: 50  # Scale down max 50% at a time
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100  # Can double capacity
        periodSeconds: 30
      - type: Pods
        value: 2  # Or add 2 pods at a time
        periodSeconds: 30
      selectPolicy: Max
```

### vLLM GPU Autoscaling

**Scale vLLM based on queue depth:**

```yaml
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-hpa
  namespace: ai-inference
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm
  minReplicas: 2
  maxReplicas: 8  # Limited by available GPUs
  metrics:
  # Scale based on queue depth (custom metric)
  - type: Pods
    pods:
      metric:
        name: vllm_queue_depth
      target:
        type: AverageValue
        averageValue: "5"  # Avg 5 requests queued per pod
  
  # Don't scale down if GPU utilization high
  - type: Pods
    pods:
      metric:
        name: gpu_utilization_percent
      target:
        type: AverageValue
        averageValue: "80"
```

**GPU node pool configuration:**

```yaml
---
# MachineSet for GPU nodes
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: gpu-a100-us-east-1a
  namespace: openshift-machine-api
spec:
  replicas: 4
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-machine-role: gpu
      machine.openshift.io/cluster-api-machine-type: gpu
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-machine-role: gpu
        machine.openshift.io/cluster-api-machine-type: gpu
    spec:
      providerSpec:
        value:
          instanceType: p4d.24xlarge  # 8x A100 GPUs
          labels:
            node-role.kubernetes.io/gpu: ""
            nvidia.com/gpu.product: A100
          taints:
          - key: nvidia.com/gpu
            value: "true"
            effect: NoSchedule

---
# vLLM deployment with GPU affinity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
  namespace: ai-inference
spec:
  template:
    spec:
      nodeSelector:
        nvidia.com/gpu.product: A100
      tolerations:
      - key: nvidia.com/gpu
        operator: Equal
        value: "true"
        effect: NoSchedule
      containers:
      - name: vllm
        resources:
          limits:
            nvidia.com/gpu: "4"  # 4 GPUs per pod
```

### Performance Tuning

**OGX connection pooling:**

```yaml
# OGX config.yaml
providers:
  inference:
    - provider_id: vllm
      provider_type: remote::vllm
      config:
        base_url: http://vllm.ai-inference.svc.cluster.local:8000
        
        # Connection pool settings
        network:
          max_connections: 100
          max_keepalive_connections: 20
          keepalive_expiry: 30
          timeout:
            connect: 5.0
            read: 60.0
            write: 60.0
            pool: 5.0
```

**Redis caching for frequent queries:**

```yaml
# OGX config.yaml
storage:
  backends:
    kv_default:
      type: kv_redis
      url: redis://redis.ai-storage.svc.cluster.local:6379
      
      # Connection pool
      max_connections: 50
      
      # Cache TTL
      default_ttl: 3600  # 1 hour
      
      # Enable caching for specific operations
      cache:
        model_metadata: true
        vector_store_metadata: true
        tool_schemas: true
```

**PostgreSQL performance:**

```yaml
---
# PostgreSQL StatefulSet with tuned settings
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: ai-storage
spec:
  serviceName: postgresql
  replicas: 1
  template:
    spec:
      containers:
      - name: postgresql
        image: postgres:16
        env:
        - name: POSTGRES_DB
          value: ogx
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: password
        
        # Performance tuning
        - name: POSTGRES_SHARED_BUFFERS
          value: "4GB"
        - name: POSTGRES_EFFECTIVE_CACHE_SIZE
          value: "12GB"
        - name: POSTGRES_WORK_MEM
          value: "64MB"
        - name: POSTGRES_MAINTENANCE_WORK_MEM
          value: "1GB"
        - name: POSTGRES_MAX_CONNECTIONS
          value: "200"
        
        resources:
          requests:
            memory: "16Gi"
            cpu: "4"
          limits:
            memory: "16Gi"
            cpu: "8"
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

### Load Testing

**Benchmark OGX throughput:**

```bash
# Install k6
oc create -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: k6-script
  namespace: ai-platform
data:
  load-test.js: |
    import http from 'k6/http';
    import { check, sleep } from 'k6';
    
    export const options = {
      stages: [
        { duration: '2m', target: 50 },   // Ramp up
        { duration: '5m', target: 50 },   // Steady state
        { duration: '2m', target: 100 },  // Spike
        { duration: '5m', target: 100 },  // Steady spike
        { duration: '2m', target: 0 },    // Ramp down
      ],
    };
    
    export default function () {
      const url = 'http://ogx-api.ai-platform.svc.cluster.local:8321/v1/chat/completions';
      const payload = JSON.stringify({
        model: 'llama-3.2-3b',
        messages: [
          { role: 'user', content: 'Hello, how are you?' }
        ],
        max_tokens: 100,
      });
      
      const params = {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': 'Bearer test-key',
        },
      };
      
      const res = http.post(url, payload, params);
      
      check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 2s': (r) => r.timings.duration < 2000,
      });
      
      sleep(1);
    }
EOF

# Run load test
oc run k6-load-test \
  --image=grafana/k6:latest \
  --restart=Never \
  --namespace=ai-platform \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "k6",
      "image": "grafana/k6:latest",
      "command": ["k6", "run", "/scripts/load-test.js"],
      "volumeMounts": [{
        "name": "script",
        "mountPath": "/scripts"
      }]
    }],
    "volumes": [{
      "name": "script",
      "configMap": {"name": "k6-script"}
    }]
  }
}'

# View results
oc logs -f k6-load-test -n ai-platform
```

---

## Integration with OpenShift Features

### OpenShift Pipelines (Tekton)

**CI/CD pipeline for OGX deployment:**

```yaml
---
# Pipeline for building and deploying OGX
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: ogx-deploy-pipeline
  namespace: ai-platform
spec:
  params:
  - name: git-url
    type: string
    default: https://github.com/ogx-ai/ogx.git
  - name: git-revision
    type: string
    default: main
  - name: image-name
    type: string
    default: ogx/distribution-starter
  - name: image-tag
    type: string
    default: latest
  
  workspaces:
  - name: shared-workspace
  
  tasks:
  # 1. Clone source
  - name: fetch-repository
    taskRef:
      name: git-clone
      kind: ClusterTask
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: url
      value: $(params.git-url)
    - name: revision
      value: $(params.git-revision)
  
  # 2. Build image
  - name: build-image
    taskRef:
      name: buildah
      kind: ClusterTask
    runAfter:
    - fetch-repository
    workspaces:
    - name: source
      workspace: shared-workspace
    params:
    - name: IMAGE
      value: image-registry.openshift-image-registry.svc:5000/ai-platform/$(params.image-name):$(params.image-tag)
    - name: DOCKERFILE
      value: ./Dockerfile
  
  # 3. Run tests
  - name: run-tests
    taskRef:
      name: pytest
    runAfter:
    - build-image
    workspaces:
    - name: source
      workspace: shared-workspace
  
  # 4. Deploy to staging
  - name: deploy-staging
    taskRef:
      name: openshift-client
      kind: ClusterTask
    runAfter:
    - run-tests
    params:
    - name: SCRIPT
      value: |
        oc set image deployment/ogx-server \
          ogx=image-registry.openshift-image-registry.svc:5000/ai-platform/$(params.image-name):$(params.image-tag) \
          -n ai-platform-staging
        oc rollout status deployment/ogx-server -n ai-platform-staging
  
  # 5. Integration tests
  - name: integration-tests
    taskRef:
      name: integration-test
    runAfter:
    - deploy-staging
  
  # 6. Deploy to production (manual approval)
  - name: deploy-production
    taskRef:
      name: openshift-client
      kind: ClusterTask
    runAfter:
    - integration-tests
    params:
    - name: SCRIPT
      value: |
        oc set image deployment/ogx-server \
          ogx=image-registry.openshift-image-registry.svc:5000/ai-platform/$(params.image-name):$(params.image-tag) \
          -n ai-platform
        oc rollout status deployment/ogx-server -n ai-platform

---
# PipelineRun (triggered manually or by webhook)
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: ogx-deploy-
  namespace: ai-platform
spec:
  pipelineRef:
    name: ogx-deploy-pipeline
  workspaces:
  - name: shared-workspace
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

### OpenShift Monitoring (Prometheus/Grafana)

**ServiceMonitor for OGX metrics:**

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ogx-metrics
  namespace: ai-platform
  labels:
    app: ogx-server
spec:
  selector:
    matchLabels:
      app: ogx-server
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics

---
# Expose metrics endpoint
apiVersion: v1
kind: Service
metadata:
  name: ogx-metrics
  namespace: ai-platform
  labels:
    app: ogx-server
spec:
  ports:
  - name: metrics
    port: 8080
    targetPort: 8080
  selector:
    app: ogx-server
```

**PrometheusRule for alerts:**

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ogx-alerts
  namespace: ai-platform
spec:
  groups:
  - name: ogx
    interval: 30s
    rules:
    # High error rate
    - alert: OGXHighErrorRate
      expr: |
        rate(ogx_requests_total{status=~"5.."}[5m]) 
        / 
        rate(ogx_requests_total[5m]) 
        > 0.05
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "OGX error rate above 5%"
        description: "{{ $value }}% of requests are failing"
    
    # High latency
    - alert: OGXHighLatency
      expr: |
        histogram_quantile(0.95, 
          rate(ogx_request_duration_seconds_bucket[5m])
        ) > 2
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "OGX p95 latency above 2s"
        description: "p95 latency is {{ $value }}s"
    
    # Quota exhaustion
    - alert: OGXTeamQuotaExhausted
      expr: |
        ogx_team_quota_used / ogx_team_quota_limit > 0.9
      for: 5m
      labels:
        severity: info
      annotations:
        summary: "Team {{ $labels.team }} approaching quota limit"
        description: "{{ $labels.team }} has used {{ $value }}% of quota"
    
    # Pod down
    - alert: OGXPodDown
      expr: |
        up{job="ogx-server"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "OGX pod is down"
        description: "{{ $labels.pod }} in {{ $labels.namespace }} is down"
```

**Grafana Dashboard ConfigMap:**

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ogx-dashboard
  namespace: openshift-config-managed
  labels:
    grafana_dashboard: "true"
data:
  ogx-dashboard.json: |
    {
      "dashboard": {
        "title": "OGX Monitoring",
        "panels": [
          {
            "title": "Request Rate",
            "targets": [{
              "expr": "rate(ogx_requests_total[5m])"
            }]
          },
          {
            "title": "Error Rate",
            "targets": [{
              "expr": "rate(ogx_requests_total{status=~\"5..\"}[5m])"
            }]
          },
          {
            "title": "Latency (p50, p95, p99)",
            "targets": [
              {
                "expr": "histogram_quantile(0.50, rate(ogx_request_duration_seconds_bucket[5m]))",
                "legendFormat": "p50"
              },
              {
                "expr": "histogram_quantile(0.95, rate(ogx_request_duration_seconds_bucket[5m]))",
                "legendFormat": "p95"
              },
              {
                "expr": "histogram_quantile(0.99, rate(ogx_request_duration_seconds_bucket[5m]))",
                "legendFormat": "p99"
              }
            ]
          },
          {
            "title": "Team Quota Usage",
            "targets": [{
              "expr": "ogx_team_quota_used / ogx_team_quota_limit"
            }]
          },
          {
            "title": "Model Usage Distribution",
            "targets": [{
              "expr": "sum by (model) (rate(ogx_requests_total[5m]))"
            }]
          }
        ]
      }
    }
```

### OpenShift Logging (EFK Stack)

**ClusterLogForwarder for OGX logs:**

```yaml
---
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: ogx-log-forwarder
  namespace: openshift-logging
spec:
  pipelines:
  - name: ogx-logs
    inputRefs:
    - application
    outputRefs:
    - elasticsearch
    - splunk
    parse: json
    labels:
      app: ogx-server
  
  outputs:
  - name: elasticsearch
    type: elasticsearch
    url: https://elasticsearch.ai-platform.svc:9200
    secret:
      name: elasticsearch-credentials
  
  - name: splunk
    type: splunk
    url: https://splunk.example.com:8088
    secret:
      name: splunk-credentials
```

**Structured logging from OGX:**

```json
{
  "timestamp": "2026-05-26T14:32:15.123Z",
  "level": "info",
  "message": "Request completed",
  "principal": "team:team-a",
  "model": "llama-3.3-70b",
  "provider": "vllm",
  "duration_ms": 1523,
  "status_code": 200,
  "input_tokens": 45,
  "output_tokens": 128,
  "request_id": "req_abc123xyz"
}
```

---

## Operational Guide

### Day 1: Initial Deployment

**Prerequisites checklist:**

```bash
# 1. Verify OpenShift cluster
oc version
# Client: 4.15.0
# Server: 4.15.12

# 2. Verify GPU Operator (if using GPUs)
oc get pods -n nvidia-gpu-operator
# NAME                                       READY   STATUS
# nvidia-cuda-validator-xxxxx                1/1     Running
# nvidia-device-plugin-daemonset-xxxxx       1/1     Running
# nvidia-driver-daemonset-xxxxx              1/1     Running

# 3. Verify storage classes
oc get storageclass
# NAME                 PROVISIONER
# gp3 (default)        ebs.csi.aws.com
# fast-ssd             ebs.csi.aws.com
# nfs-storage          nfs.io

# 4. Create namespaces
oc create namespace ai-platform
oc create namespace ai-inference
oc create namespace ai-storage

# 5. Apply RBAC
oc create serviceaccount ogx-server -n ai-platform
oc adm policy add-scc-to-user anyuid -z ogx-server -n ai-platform

# 6. Deploy storage layer
oc apply -f postgresql-statefulset.yaml
oc apply -f redis-deployment.yaml

# Wait for storage to be ready
oc wait --for=condition=ready pod -l app=postgresql -n ai-storage --timeout=300s

# 7. Deploy inference layer
oc apply -f vllm-deployment.yaml
oc apply -f ollama-deployment.yaml

# Wait for models to load (can take 10+ minutes)
oc wait --for=condition=ready pod -l app=vllm -n ai-inference --timeout=600s

# 8. Deploy OGX
oc apply -f ogx-configmap.yaml
oc apply -f ogx-deployment.yaml
oc apply -f ogx-service.yaml
oc apply -f ogx-route.yaml

# Wait for OGX to be ready
oc wait --for=condition=ready pod -l app=ogx-server -n ai-platform --timeout=300s

# 9. Verify deployment
oc get pods -n ai-platform
oc get pods -n ai-inference
oc get pods -n ai-storage

# 10. Test OGX endpoint
OGX_URL=$(oc get route ogx-api -n ai-platform -o jsonpath='{.spec.host}')
curl -k https://$OGX_URL/health
# {"status": "healthy"}
```

### Day 2: Operations

**Health checks:**

```bash
# Check OGX health
oc exec -n ai-platform deploy/ogx-server -- curl localhost:8321/health

# Check backend connectivity
oc exec -n ai-platform deploy/ogx-server -- \
  curl http://vllm.ai-inference.svc.cluster.local:8000/health

# Check database connectivity
oc exec -n ai-platform deploy/ogx-server -- \
  psql -h postgresql.ai-storage.svc.cluster.local -U ogx_user -d ogx -c "SELECT 1;"
```

**Log aggregation:**

```bash
# Stream OGX logs
oc logs -f deploy/ogx-server -n ai-platform

# Search logs for errors
oc logs deploy/ogx-server -n ai-platform | grep ERROR

# Get logs from all replicas
oc logs -l app=ogx-server -n ai-platform --tail=100

# Export logs to file
oc logs deploy/ogx-server -n ai-platform --since=24h > ogx-logs-24h.log
```

**Metrics:**

```bash
# Get Prometheus metrics
oc exec -n ai-platform deploy/ogx-server -- curl localhost:8080/metrics

# Query specific metric
oc exec -n ai-platform deploy/ogx-server -- \
  curl -s localhost:8080/metrics | grep ogx_requests_total

# Port-forward to view locally
oc port-forward -n ai-platform svc/ogx-metrics 8080:8080
# Then: curl localhost:8080/metrics
```

### Backup & Restore

**Database backups:**

```bash
# Create backup job
oc create job --from=cronjob/postgresql-backup backup-manual -n ai-storage

# Verify backup
oc logs job/backup-manual -n ai-storage

# List backups
oc exec -n ai-storage deploy/postgresql -- \
  ls -lh /backups/

# Restore from backup
oc exec -n ai-storage deploy/postgresql -- \
  psql -U ogx_user -d ogx < /backups/ogx-2026-05-26.sql
```

**Configuration backups:**

```bash
# Backup all OGX configs
oc get configmap ogx-config -n ai-platform -o yaml > ogx-config-backup.yaml
oc get secret postgresql-credentials -n ai-platform -o yaml > secrets-backup.yaml

# Backup entire namespace
oc get all -n ai-platform -o yaml > ai-platform-backup.yaml
```

### Upgrades

**Rolling upgrade of OGX:**

```bash
# 1. Check current version
oc get deploy ogx-server -n ai-platform -o jsonpath='{.spec.template.spec.containers[0].image}'

# 2. Update image
oc set image deployment/ogx-server \
  ogx=docker.io/ogx/distribution-starter:1.0.4 \
  -n ai-platform

# 3. Watch rollout
oc rollout status deployment/ogx-server -n ai-platform

# 4. Verify new version
oc exec -n ai-platform deploy/ogx-server -- ogx --version

# 5. If problems, rollback
oc rollout undo deployment/ogx-server -n ai-platform
```

**Zero-downtime upgrade strategy:**

```yaml
---
# Deployment with RollingUpdate strategy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ogx-server
  namespace: ai-platform
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max 1 extra pod during upgrade
      maxUnavailable: 0  # Always keep all pods available
  template:
    spec:
      containers:
      - name: ogx
        image: docker.io/ogx/distribution-starter:1.0.4
        livenessProbe:
          httpGet:
            path: /health
            port: 8321
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8321
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3

# Upgrade process:
# 1. New pod created (total: 4 pods)
# 2. Wait for new pod ready
# 3. Old pod terminated (back to 3 pods)
# 4. Repeat until all pods upgraded
```

### Disaster Recovery

**Scenario: Complete cluster failure**

```bash
# 1. Rebuild cluster (OpenShift installation)

# 2. Restore storage layer
oc apply -f postgresql-statefulset.yaml
oc apply -f redis-deployment.yaml

# Wait for storage pods
oc wait --for=condition=ready pod -l app=postgresql -n ai-storage --timeout=300s

# 3. Restore database from backup
oc cp ogx-backup-2026-05-26.sql \
  postgresql-0:/tmp/restore.sql \
  -n ai-storage

oc exec -n ai-storage postgresql-0 -- \
  psql -U ogx_user -d ogx -f /tmp/restore.sql

# 4. Restore model weights to PVC
oc apply -f model-storage-pvc.yaml

oc run model-restore \
  --image=alpine \
  --restart=Never \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "restore",
      "image": "alpine",
      "command": ["sleep", "3600"],
      "volumeMounts": [{
        "name": "models",
        "mountPath": "/models"
      }]
    }],
    "volumes": [{
      "name": "models",
      "persistentVolumeClaim": {"claimName": "model-storage"}
    }]
  }
}'

# Copy models from backup location
oc cp models-backup.tar.gz model-restore:/tmp/
oc exec model-restore -- tar -xzf /tmp/models-backup.tar.gz -C /models/

# 5. Deploy inference layer
oc apply -f vllm-deployment.yaml
oc apply -f ollama-deployment.yaml

# 6. Deploy OGX layer
oc apply -f ogx-configmap.yaml
oc apply -f ogx-deployment.yaml
oc apply -f ogx-service.yaml
oc apply -f ogx-route.yaml

# 7. Verify everything works
curl -k https://$(oc get route ogx-api -n ai-platform -o jsonpath='{.spec.host}')/health
```

---

## Troubleshooting

### Common Issues

#### Issue 1: OGX pods not starting

**Symptom:**
```bash
oc get pods -n ai-platform
# NAME                         READY   STATUS             RESTARTS
# ogx-server-xxxxx             0/1     CrashLoopBackOff   5
```

**Diagnosis:**
```bash
# Check pod logs
oc logs ogx-server-xxxxx -n ai-platform

# Common causes:
# - Database connection failure
# - Invalid configuration
# - Missing secrets
```

**Solutions:**

```bash
# 1. Verify database connectivity
oc exec -n ai-platform deploy/ogx-server -- \
  nc -zv postgresql.ai-storage.svc.cluster.local 5432

# 2. Check configuration
oc get configmap ogx-config -n ai-platform -o yaml

# 3. Verify secrets exist
oc get secret postgresql-credentials -n ai-platform

# 4. Check events
oc describe pod ogx-server-xxxxx -n ai-platform | grep Events -A 20
```

#### Issue 2: High latency / slow responses

**Symptom:**
```bash
# Responses taking 10+ seconds
curl -w "@curl-format.txt" -o /dev/null -s \
  https://ogx-api.apps.cluster.example.com/v1/chat/completions
# time_total: 12.543s
```

**Diagnosis:**
```bash
# Check OGX metrics
oc exec -n ai-platform deploy/ogx-server -- \
  curl -s localhost:8080/metrics | grep duration

# Check backend health
oc exec -n ai-inference deploy/vllm -- \
  curl -s localhost:8000/v1/models
```

**Solutions:**

```bash
# 1. Check if vLLM queue is backed up
oc logs -n ai-inference deploy/vllm | grep "queue"

# 2. Scale vLLM if needed
oc scale deployment vllm -n ai-inference --replicas=4

# 3. Check network latency
oc exec -n ai-platform deploy/ogx-server -- \
  ping -c 5 vllm.ai-inference.svc.cluster.local

# 4. Enable connection pooling (config.yaml)
# See Performance Tuning section
```

#### Issue 3: GPU not being used

**Symptom:**
```bash
# vLLM pod scheduled but no GPU utilization
nvidia-smi
# No processes using GPU
```

**Diagnosis:**
```bash
# Check GPU allocation
oc describe pod vllm-xxxxx -n ai-inference | grep nvidia.com/gpu

# Check GPU Operator
oc get pods -n nvidia-gpu-operator
```

**Solutions:**

```bash
# 1. Verify GPU request in deployment
oc get deployment vllm -n ai-inference -o yaml | grep -A 3 resources

# Should see:
#   resources:
#     limits:
#       nvidia.com/gpu: "4"

# 2. Check node labels
oc get nodes --show-labels | grep nvidia

# 3. Verify GPU plugin
oc logs -n nvidia-gpu-operator \
  -l app=nvidia-device-plugin-daemonset

# 4. Check pod placement
oc get pod vllm-xxxxx -n ai-inference -o wide
# Verify it's on a GPU node

# 5. Restart pod if needed
oc delete pod vllm-xxxxx -n ai-inference
```

#### Issue 4: Team hitting quota limits

**Symptom:**
```bash
# Application receives 429 Too Many Requests
curl -i https://ogx-api.apps.cluster.example.com/v1/chat/completions \
  -H "Authorization: Bearer team-a-key"
# HTTP/1.1 429 Too Many Requests
```

**Diagnosis:**
```bash
# Check team quota usage
oc exec -n ai-platform deploy/ogx-server -- \
  ogx admin quota-status --team team-a

# Output:
# Team: team-a
# Quota: 10000 requests/day
# Used: 10245 requests
# Remaining: 0
```

**Solutions:**

```bash
# Option 1: Increase quota (config.yaml)
oc edit configmap ogx-config -n ai-platform
# Change quota.requests_per_day: 20000

# Restart OGX to apply
oc rollout restart deployment ogx-server -n ai-platform

# Option 2: Reset quota manually (emergency)
oc exec -n ai-platform deploy/ogx-server -- \
  ogx admin quota-reset --team team-a

# Option 3: Upgrade team tier (long-term)
```

### Debugging Tools

**Interactive debugging:**

```bash
# Shell into OGX pod
oc rsh -n ai-platform deploy/ogx-server

# Inside pod:
# - Check config: cat /etc/ogx/config.yaml
# - Test backends: curl http://vllm.ai-inference.svc.cluster.local:8000/health
# - Check logs: tail -f /var/log/ogx/server.log
# - Query database: psql -h postgresql.ai-storage.svc -U ogx_user -d ogx
```

**Network debugging:**

```bash
# Create debug pod
oc run debug-pod \
  --image=nicolaka/netshoot \
  --restart=Never \
  -n ai-platform \
  -- sleep 3600

# Inside debug pod:
oc exec -it debug-pod -n ai-platform -- bash

# Test DNS
nslookup vllm.ai-inference.svc.cluster.local

# Test connectivity
curl http://vllm.ai-inference.svc.cluster.local:8000/health

# Check routes
ip route

# Capture traffic
tcpdump -i any port 8321 -w /tmp/ogx-traffic.pcap
```

**Performance profiling:**

```bash
# CPU profiling
oc exec -n ai-platform deploy/ogx-server -- \
  py-spy record -o profile.svg -- python -m ogx.server

# Memory profiling
oc exec -n ai-platform deploy/ogx-server -- \
  memory_profiler python -m ogx.server

# Request tracing (if OpenTelemetry enabled)
oc port-forward -n ai-platform svc/jaeger-query 16686:16686
# Open http://localhost:16686 in browser
```

---

## Migration Strategies

### Migrating from Direct Provider Access to OGX

**Scenario:** Team currently calls vLLM directly, wants to migrate to OGX

**Before (direct vLLM):**
```python
from openai import OpenAI

client = OpenAI(
    base_url="http://vllm.ai-inference.svc.cluster.local:8000/v1",
    api_key="fake"
)
```

**After (via OGX):**
```python
from openai import OpenAI

client = OpenAI(
    base_url="http://ogx-api.ai-platform.svc.cluster.local:8321/v1",
    api_key="team-a-key"  # Now tracked by OGX
)
```

**Migration steps:**

```bash
# 1. Deploy OGX alongside existing vLLM (no changes yet)
oc apply -f ogx-deployment.yaml

# 2. Configure OGX to route to existing vLLM
# (config.yaml already points to vllm.ai-inference.svc)

# 3. Update one application as canary
oc set env deployment/chatbot \
  OGX_BASE_URL=http://ogx-api.ai-platform.svc.cluster.local:8321/v1 \
  -n team-a

# 4. Monitor for issues
oc logs -f deployment/chatbot -n team-a

# 5. If successful, roll out to remaining apps
oc set env deployment/app-2 OGX_BASE_URL=... -n team-a
oc set env deployment/app-3 OGX_BASE_URL=... -n team-a

# 6. After all migrations, enforce NetworkPolicy
# (only OGX can access vLLM, teams cannot access directly)
```

**Rollback plan:**
```bash
# Revert to direct vLLM access
oc set env deployment/chatbot \
  OGX_BASE_URL=http://vllm.ai-inference.svc.cluster.local:8000/v1 \
  -n team-a
```

### Migrating from Cloud to On-Prem

**Scenario:** Team using OpenAI cloud, migrating to on-prem OGX + vLLM

**Before:**
```python
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
# Calls api.openai.com
```

**After:**
```python
from openai import OpenAI

client = OpenAI(
    base_url="http://ogx-api.ai-platform.svc.cluster.local:8321/v1",
    api_key="team-a-key"
)
# Same code, different endpoint
```

**Migration considerations:**

| Aspect | Cloud (OpenAI) | On-Prem (OGX) | Migration Impact |
|--------|----------------|---------------|------------------|
| **Models** | `gpt-4`, `gpt-3.5-turbo` | `llama-3.3-70b`, `llama-3.2-3b` | Model names change, quality may differ |
| **Cost** | Per-token pricing | Fixed infra cost | Billing model changes |
| **Latency** | 500-2000ms | 200-800ms (lower, but variable) | Faster on-prem |
| **Availability** | 99.9% SLA | Self-managed (depends on ops) | Responsibility shifts |
| **Features** | Latest OpenAI features | May lag behind | Feature parity check needed |
| **Data location** | OpenAI cloud | Your cluster | Compliance benefit |

**Migration steps:**

```bash
# Phase 1: Parallel run (shadow traffic)
# - Keep cloud as primary
# - Send duplicate requests to OGX for testing
# - Compare responses

# Phase 2: Canary
# - Route 10% of traffic to OGX
# - Monitor quality, errors, latency
# - Increase to 50% if successful

# Phase 3: Full migration
# - Route 100% to OGX
# - Keep cloud credentials for emergency fallback
# - Monitor for 2 weeks

# Phase 4: Decomm cloud
# - Remove OPENAI_API_KEY from configs
# - Cancel subscription
```

---

## Best Practices

### Security Best Practices

1. **Least Privilege RBAC**
   - Teams can only access their own namespace
   - ServiceAccounts scoped to specific APIs
   - No cluster-admin for AI workloads

2. **Network Segmentation**
   - NetworkPolicies isolate namespaces
   - Teams cannot bypass OGX to access backends directly
   - Deny all egress except to OGX

3. **Secrets Management**
   - Use External Secrets Operator for production
   - Rotate API keys regularly
   - Never commit secrets to git

4. **Audit Everything**
   - Enable OGX audit logging
   - Forward logs to SIEM
   - Retain logs for compliance period

5. **TLS Everywhere**
   - Use Service Mesh for automatic mTLS
   - Or configure TLS manually for all services
   - Rotate certificates regularly

### Performance Best Practices

1. **Right-size Replicas**
   - OGX: Start with 3, scale based on request rate
   - vLLM: Start with 2, scale based on queue depth
   - PostgreSQL: Usually 1 (or 3 for HA)

2. **Resource Limits**
   - Always set both requests and limits
   - Leave headroom for spikes (limits 2x requests)
   - Monitor actual usage, adjust over time

3. **Connection Pooling**
   - Configure OGX connection pools
   - PostgreSQL max_connections: 200
   - Redis: max_connections: 50

4. **Caching Strategy**
   - Cache model metadata in Redis (1 hour TTL)
   - Cache tool schemas in Redis (24 hour TTL)
   - Don't cache inference results (always fresh)

5. **GPU Utilization**
   - Monitor GPU usage (should be >80%)
   - If <50%, reduce replicas or model size
   - Use tensor parallelism for large models

### Operational Best Practices

1. **Monitoring**
   - Set up Prometheus alerts (see examples)
   - Create Grafana dashboards
   - Monitor: latency, error rate, quota usage, GPU util

2. **Backup Strategy**
   - Daily PostgreSQL backups
   - Weekly model weights backups
   - Store backups off-cluster (S3, etc.)
   - Test restore procedure quarterly

3. **Upgrade Strategy**
   - Test in staging first
   - Use rolling updates (maxUnavailable: 0)
   - Keep previous version image for rollback
   - Monitor post-upgrade for 24h

4. **Capacity Planning**
   - Track growth trends (requests/day)
   - Plan GPU additions 3 months ahead
   - Monitor storage usage (90% triggers warning)
   - Forecast costs based on usage trends

5. **Documentation**
   - Document architecture decisions (ADRs)
   - Runbooks for common operations
   - Disaster recovery procedures
   - Team onboarding guide

### Cost Optimization Best Practices

1. **Right-size Models**
   - Use smallest model that meets quality bar
   - llama-3.2-3b for simple tasks
   - llama-3.3-70b for complex reasoning

2. **Quota Management**
   - Set conservative quotas initially
   - Increase based on demonstrated need
   - Review quota usage monthly

3. **GPU Efficiency**
   - Pack multiple small models on one GPU
   - Use CPU for embeddings
   - Turn off dev GPUs at night (CronJobs)

4. **Storage Optimization**
   - Use object storage for files (cheaper than PVCs)
   - Compress old logs before archiving
   - Delete test data after 30 days

5. **Cloud Bursting**
   - Run baseline on-prem
   - Burst to cloud for spikes (OGX config change only)
   - Track costs per environment

---

## Summary

**OGX on OpenShift provides:**

✅ **Multi-tenancy** - Infrastructure (OpenShift) + AI workloads (OGX)  
✅ **Unified API** - OpenAI, Anthropic, Google SDKs against any backend  
✅ **Governance** - Centralized quotas, access control, cost tracking  
✅ **Security** - NetworkPolicies, mTLS, audit logging, data sovereignty  
✅ **Scalability** - HPA for OGX and backends, GPU scheduling  
✅ **Observability** - Prometheus metrics, Grafana dashboards, OpenTelemetry  
✅ **Operational maturity** - CI/CD, monitoring, backup/restore, disaster recovery  

**Key decision points:**

| Requirement | Use OGX on OpenShift | Use Direct Access |
|-------------|----------------------|-------------------|
| Multi-team platform | ✅ Yes | ❌ No |
| Governance needed | ✅ Yes | ❌ No |
| Complex workflows (RAG, tools) | ✅ Yes | ⚠️ Maybe |
| Single team, simple use case | ⚠️ Maybe | ✅ Yes |
| Want provider flexibility | ✅ Yes | ❌ No |
| Limited ops capacity | ❌ No | ✅ Yes |

**Next steps:**

1. Review decision framework (matches your requirements?)
2. Deploy proof-of-concept in non-prod cluster
3. Benchmark performance vs. direct access
4. Pilot with one team
5. Roll out platform-wide
6. Establish operational runbooks

---

## Additional Resources

- **OGX Documentation:** https://ogx-ai.github.io/docs
- **OpenShift Documentation:** https://docs.openshift.com
- **NVIDIA GPU Operator:** https://docs.nvidia.com/datacenter/cloud-native/gpu-operator
- **OpenShift AI:** https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed
- **Service Mesh:** https://docs.openshift.com/container-platform/latest/service_mesh

**Community:**
- OGX Discord: https://discord.gg/ZAFjsrcw
- OGX GitHub: https://github.com/ogx-ai/ogx
- OpenShift Community: https://www.openshift.com/community

---

**Document Version:** 1.0  
**Last Updated:** 2026-05-26  
**Maintained By:** OGX Community
