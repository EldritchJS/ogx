# OGX Deployment Guide for Barcelona Cluster

**Generated:** 2026-05-28  
**Cluster:** barcelona.nerc.mghpcc.org  
**Purpose:** Deploy OGX for multi-tenant research environment with healthcare/medical data compliance

---

## Table of Contents

1. [Cluster Overview](#cluster-overview)
2. [Architecture for Barcelona](#architecture-for-barcelona)
3. [Prerequisites](#prerequisites)
4. [Quick Start Deployment](#quick-start-deployment)
5. [Multi-Tenancy Setup](#multi-tenancy-setup)
6. [Data Isolation & Security](#data-isolation--security)
7. [Usage Tracking Configuration](#usage-tracking-configuration)
8. [HIPAA/Healthcare Compliance](#hipaahealthcare-compliance)
9. [GPU Scheduling](#gpu-scheduling)
10. [Monitoring & Operations](#monitoring--operations)
11. [Researcher Onboarding](#researcher-onboarding)
12. [Troubleshooting](#troubleshooting)

---

## Cluster Overview

### Barcelona Cluster Specs

**Infrastructure:**
- **OpenShift Version:** 4.20.10 (Kubernetes v1.33.6)
- **Control Plane:** 3 nodes
- **Worker Nodes:** 8 nodes
  - 3 CPU-only workers
  - **5 H100 GPU nodes** (4x H100-80GB per node = 20 H100 GPUs total)
- **Ingress Domain:** `*.apps.barcelona.nerc.mghpcc.org`

**Storage Classes:**
- `ocs-external-storagecluster-ceph-rbd` (default) - Ceph block storage
- `nfs-csi` - NFS storage
- `openshift-storage.noobaa.io` - S3-compatible object storage

**Pre-Installed Operators:**
- ✅ NVIDIA GPU Operator (running, validated)
- ✅ External Secrets Operator
- ✅ OpenShift GitOps (ArgoCD)
- ✅ OpenShift Pipelines (Tekton)
- ✅ Cert Manager
- ✅ KEDA (Custom Metrics Autoscaler)
- ✅ Grafana Operator
- ✅ Loki (logging)

**Existing Multi-Tenancy Pattern:**
- Projects: `researcher-a`, `researcher-b` (already exist)
- Pattern: Separate namespace per researcher/team

---

## Architecture for Barcelona

### Deployment Model

```text
┌────────────────────────────────────────────────────────────────┐
│  Researcher Namespaces (Isolated)                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐ │
│  │ researcher-a     │  │ researcher-b     │  │ researcher-c │ │
│  │ - App pods       │  │ - App pods       │  │ - App pods   │ │
│  │ - Data isolated  │  │ - Data isolated  │  │ - Isolated   │ │
│  │ - Quota: 1k/day  │  │ - Quota: 5k/day  │  │ - Quota TBD  │ │
│  └──────────────────┘  └──────────────────┘  └──────────────┘ │
│         │                      │                      │         │
│         └──────────────────────┴──────────────────────┘         │
│                               ↓                                 │
│              All traffic goes through OGX (central choke point) │
└────────────────────────────────────────────────────────────────┘
                                ↓
┌────────────────────────────────────────────────────────────────┐
│  OGX Platform Namespace: ogx-platform                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ OGX Server (managed by operator, 3 replicas)             │  │
│  │ - Multi-SDK API (OpenAI, Anthropic, Google)              │  │
│  │ - Per-researcher quotas & access control                 │  │
│  │ - Usage tracking (all requests logged)                   │  │
│  │ - Data isolation enforcement                             │  │
│  │ - Declarative OGXDistribution custom resource            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                               ↓                                 │
│              Routes to inference & storage backends             │
└────────────────────────────────────────────────────────────────┘
                                ↓
        ┌───────────────────────┴─────────────────────┐
        │                                             │
        ↓                                             ↓
┌──────────────────────┐                  ┌──────────────────────┐
│ Inference Namespace  │                  │ Storage Namespace    │
│ ogx-inference        │                  │ ogx-storage          │
│                      │                  │                      │
│ ┌────────────────┐   │                  │ ┌────────────────┐   │
│ │ vLLM (H100s)   │   │                  │ │ PostgreSQL     │   │
│ │ - Llama models │   │                  │ │ + pgvector     │   │
│ │ - 4x H100 GPU  │   │                  │ │                │   │
│ └────────────────┘   │                  │ │ - User data    │   │
│                      │                  │ │ - Vector store │   │
│ ┌────────────────┐   │                  │ └────────────────┘   │
│ │ Ollama (CPU)   │   │                  │                      │
│ │ - Small models │   │                  │ ┌────────────────┐   │
│ │ - Dev/testing  │   │                  │ │ Redis          │   │
│ └────────────────┘   │                  │ │ - OGX cache    │   │
│                      │                  │ └────────────────┘   │
└──────────────────────┘                  └──────────────────────┘
```

**Key Design Decisions:**
1. **Operator-Managed OGX** - OGX deployed via Kubernetes operator for declarative management, automatic config reloading, and simplified upgrades
2. **Central OGX Instance** - Single OGX deployment serves all researchers
3. **Namespace Isolation** - Each researcher gets own namespace for apps/data
4. **NetworkPolicies** - Researchers can only talk to OGX, not each other
5. **Per-Researcher Quotas** - Usage limits enforced by OGX
6. **Audit Trail** - All requests logged with researcher identity

---

## Prerequisites

### Access Verification

```bash
# Verify you're logged in as cluster admin
oc whoami
# Should output: EldritchJS

# Verify cluster admin permissions
oc auth can-i '*' '*'
# Should output: yes

# Verify GPU nodes are ready
oc get nodes -o json | jq -r '.items[] | select(.status.capacity."nvidia.com/gpu" != null) | "\(.metadata.name): \(.status.capacity."nvidia.com/gpu") GPUs"'
# Should show 5 nodes with 4 H100-80GB GPUs each (20 total)
```

### Resource Planning

**OGX Platform (ogx-platform namespace):**
- **OGX Server:** 3 replicas, 2Gi RAM each = 6Gi total
- **PostgreSQL:** 1 replica, 16Gi RAM
- **Redis:** 1 replica, 2Gi RAM
- **Total:** ~25Gi RAM, 6 CPU cores

**Inference (ogx-inference namespace):**
- **vLLM:** 1-5 pods with 4x H100-80GB GPUs each (can scale across all 5 GPU nodes)
- **Ollama:** 3 replicas (CPU), 4Gi RAM each = 12Gi total
- **Total:** Up to 20 H100 GPUs available, scalable based on demand

**Storage (ogx-storage namespace):**
- **PostgreSQL PVC:** 100Gi (Ceph RBD)
- **Redis PVC:** 10Gi (Ceph RBD)
- **Model weights:** 500Gi (Ceph RBD, ReadWriteMany NFS if needed)

---

## GPU Cost Awareness

**IMPORTANT: Only vLLM claims GPU resources. All other OGX components run on CPU-only nodes.**

### Which Components Use GPUs?

| Component | GPU Usage | Cost Impact |
|-----------|-----------|-------------|
| **vLLM** | ✅ **YES** - Requests 4 GPUs per pod | **Billed for GPU time** |
| OGX Server | ❌ No - CPU-only | Free (CPU nodes) |
| Ollama | ❌ No - CPU-only | Free (CPU nodes) |
| PostgreSQL | ❌ No - CPU-only | Free (CPU nodes) |
| Redis | ❌ No - CPU-only | Free (CPU nodes) |

**vLLM deployment claims:**
- 4 GPUs per replica
- If you deploy 1 vLLM replica = 4 H100s allocated = **billed**
- If you deploy 5 vLLM replicas = 20 H100s allocated = **billed**

### Testing OGX Without GPU Costs

**You can fully test OGX features without deploying vLLM:**

1. **Skip Step 3** (Deploy Inference Layer) in the Quick Start
2. Deploy only OGX + PostgreSQL + Redis + Ollama (all CPU-only)
3. Test with Ollama CPU inference for small models

**What you can test without GPUs:**
- ✅ Multi-tenancy (researcher namespaces, access control)
- ✅ Quota enforcement
- ✅ Usage tracking and audit logs
- ✅ API compatibility (OpenAI, Anthropic SDKs)
- ✅ Responses API orchestration
- ✅ Vector storage (pgvector)
- ✅ Small model inference via Ollama (slower, but functional)

**When to deploy vLLM (incurs GPU costs):**
- Production workloads requiring high throughput
- Large model inference (70B+ parameter models)
- Performance benchmarking
- After validating OGX setup on CPU-only components

### Verify No GPUs Are Allocated

```bash
# Check if any pods are using GPUs
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.limits."nvidia.com/gpu" != null) | 
  "\(.metadata.namespace)/\(.metadata.name): \(.spec.containers[0].resources.limits."nvidia.com/gpu") GPUs"'

# Should return empty if vLLM is not deployed
# If vLLM is deployed, you'll see:
# ogx-inference/vllm-xxxxx: 4 GPUs

# Check GPU node allocation
oc describe nodes | grep -A 5 "Allocated resources" | grep nvidia

# View GPU quota usage by namespace
oc get resourcequota --all-namespaces
```

### Cost-Saving Deployment Path

**Phase 1: Free tier (CPU-only testing)**
```bash
# Deploy: OGX + PostgreSQL + Redis + Ollama
# Skip vLLM deployment entirely
# Use Ollama with llama-3.2-3b for testing
# Cost: $0 GPU charges
```

**Phase 2: GPU deployment (when ready)**
```bash
# Deploy vLLM with 1 replica (4 GPUs)
# Test production workloads
# Cost: 4 H100 GPUs charged
```

**Phase 3: Production scale (high throughput)**
```bash
# Scale vLLM to 5 replicas (20 GPUs)
# Cost: 20 H100 GPUs charged
```

---

## Quick Start Deployment

### Step 1: Create Namespaces

```bash
# Core OGX namespaces
oc new-project ogx-platform --description="OGX API Gateway"
oc new-project ogx-inference --description="Inference backends (vLLM, Ollama)"
oc new-project ogx-storage --description="Storage layer (PostgreSQL, Redis)"

# Verify
oc get projects | grep ogx-
```

### Step 2: Deploy Storage Layer

**PostgreSQL with pgvector:**

```bash
# Create PostgreSQL secret
oc create secret generic postgresql-credentials \
  -n ogx-storage \
  --from-literal=username=ogx_user \
  --from-literal=password=$(openssl rand -base64 32) \
  --from-literal=database=ogx

# Deploy PostgreSQL
cat <<EOF | oc apply -f -
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: ogx-storage
spec:
  serviceName: postgresql
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: ankane/pgvector:v0.7.0
        ports:
        - containerPort: 5432
          name: postgresql
        env:
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: database
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
        - name: POSTGRES_SHARED_BUFFERS
          value: "4GB"
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
      storageClassName: ocs-external-storagecluster-ceph-rbd
      resources:
        requests:
          storage: 100Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: ogx-storage
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgresql
EOF

# Wait for PostgreSQL to be ready
oc wait --for=condition=ready pod -l app=postgresql -n ogx-storage --timeout=300s
```

**Redis:**

```bash
cat <<EOF | oc apply -f -
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: ogx-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "2Gi"
            cpu: "2"
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: ogx-storage
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
EOF
```

### Step 3: Deploy Inference Layer

> **⚠️ GPU COST WARNING**  
> **This step deploys vLLM which will claim 4 H100 GPUs and incur GPU charges.**
>
> You can skip this step for initial testing and use only Ollama (CPU-only) instead.  
> See [GPU Cost Awareness](#gpu-cost-awareness) section for details.
>
> To proceed without GPU costs:
> 1. Skip vLLM deployment (this entire section)
> 2. Deploy only Ollama (CPU-only, shown below)
> 3. Update OGX config to use only `ollama` provider

#### Option A: Deploy vLLM (Uses 4 GPUs - Incurs Costs)

**Download and prepare Llama model:**

```bash
# Create PVC for model storage (using NFS for ReadWriteMany)
cat <<EOF | oc apply -f -
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage
  namespace: ogx-inference
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 500Gi
EOF

# Create a job to download models (requires internet access)
cat <<EOF | oc apply -f -
---
apiVersion: batch/v1
kind: Job
metadata:
  name: model-download
  namespace: ogx-inference
spec:
  template:
    spec:
      containers:
      - name: download
        image: python:3.12
        command:
        - /bin/bash
        - -c
        - |
          pip install huggingface-hub
          huggingface-cli download meta-llama/Llama-3.3-70B-Instruct \
            --local-dir /models/llama-3.3-70b \
            --local-dir-use-symlinks False
        volumeMounts:
        - name: models
          mountPath: /models
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: model-storage
      restartPolicy: Never
  backoffLimit: 3
EOF

# Monitor download (this will take a while - ~140GB model)
oc logs -f job/model-download -n ogx-inference
```

**Deploy vLLM with H100 GPUs:**

```bash
cat <<EOF | oc apply -f -
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
  namespace: ogx-inference
  labels:
    app: vllm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm
  template:
    metadata:
      labels:
        app: vllm
    spec:
      nodeSelector:
        node-role.kubernetes.io/h100: ""
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        command:
        - python3
        - -m
        - vllm.entrypoints.openai.api_server
        args:
        - --model=/models/llama-3.3-70b
        - --served-model-name=llama-3.3-70b
        - --host=0.0.0.0
        - --port=8000
        - --tensor-parallel-size=4
        - --max-model-len=8192
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: CUDA_VISIBLE_DEVICES
          value: "0,1,2,3"
        resources:
          limits:
            nvidia.com/gpu: "4"
          requests:
            nvidia.com/gpu: "4"
            memory: "400Gi"
            cpu: "48"
        volumeMounts:
        - name: models
          mountPath: /models
          readOnly: true
        - name: shm
          mountPath: /dev/shm
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 300
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 10
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: model-storage
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 200Gi
---
apiVersion: v1
kind: Service
metadata:
  name: vllm
  namespace: ogx-inference
spec:
  type: ClusterIP
  ports:
  - port: 8000
    targetPort: 8000
  selector:
    app: vllm
EOF

# Wait for vLLM to be ready (this takes 5-10 minutes to load model)
oc logs -f deployment/vllm -n ogx-inference

# Once you see "Uvicorn running on http://0.0.0.0:8000", it's ready
```

#### Option B: Deploy Ollama Only (CPU-Only - No GPU Costs)

> **✅ NO GPU COSTS**  
> Ollama runs on CPU-only nodes and does not claim any GPU resources.
> Perfect for testing OGX features, multi-tenancy, and small model inference.

**Deploy Ollama:**

```bash
cat <<EOF | oc apply -f -
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: ogx-inference
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
      - name: ollama
        image: ollama/ollama:latest
        ports:
        - containerPort: 11434
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "4Gi"
            cpu: "4"
        volumeMounts:
        - name: data
          mountPath: /root/.ollama
      volumes:
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: ogx-inference
spec:
  type: ClusterIP
  ports:
  - port: 11434
    targetPort: 11434
  selector:
    app: ollama
EOF

# Pull a small model into Ollama
oc exec -n ogx-inference deployment/ollama -- ollama pull llama3.2:3b
```

### Step 4a: Install OGX Kubernetes Operator

The OGX Kubernetes Operator manages OGX deployments declaratively using the `OGXDistribution` custom resource.

```bash
# Install the operator from the latest release
oc apply -f https://raw.githubusercontent.com/ogx-ai/ogx-k8s-operator/main/release/operator.yaml

# Verify operator is running
oc get pods -n ogx-k8s-operator-system

# Wait for operator to be ready
oc wait --for=condition=ready pod -l control-plane=controller-manager -n ogx-k8s-operator-system --timeout=300s
```

**What the operator provides:**
- Declarative `OGXDistribution` custom resource
- Automatic Deployment, Service, and PVC management
- Config hot-reloading via ConfigMap updates
- Upgrade management for OGX versions

### Step 4b: Deploy OGX Using Operator

**Create OGX ConfigMap:**

> **Configuration Note:**
> - If you deployed **vLLM** (Option A): Use the config below as-is (includes both vLLM and Ollama providers)
> - If you deployed **Ollama only** (Option B): Remove the `vllm` provider section and keep only `ollama`

```bash
cat <<EOF | oc apply -f -
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ogx-config
  namespace: ogx-platform
data:
  config.yaml: |
    version: 2
    distro_name: barcelona-research
    
    apis:
      - inference
      - responses
      - vector_io
      - files
      - tool_runtime
    
    providers:
      inference:
        # ⚠️ vLLM provider - REMOVE this section if you skipped vLLM deployment (Option B)
        - provider_id: vllm
          provider_type: remote::vllm
          config:
            base_url: http://vllm.ogx-inference.svc.cluster.local:8000/v1
            max_tokens: 8192
        
        # ✅ Ollama provider - CPU-only, no GPU costs
        - provider_id: ollama
          provider_type: remote::ollama
          config:
            base_url: http://ollama.ogx-inference.svc.cluster.local:11434
      
      vector_io:
        - provider_id: pgvector
          provider_type: remote::pgvector
          config:
            host: postgresql.ogx-storage.svc.cluster.local
            port: 5432
            db: ogx
            user: ogx_user
            password: \${env.POSTGRES_PASSWORD}
            distance_metric: COSINE
      
      files:
        - provider_id: builtin-files
          provider_type: inline::localfs
          config:
            storage_dir: /data/ogx/files
            metadata_store:
              table_name: files_metadata
              backend: sql_default
      
      responses:
        - provider_id: builtin
          provider_type: inline::builtin
          config:
            persistence:
              responses:
                table_name: responses
                backend: sql_default
                max_write_queue_size: 10000
                num_writers: 4
      
      tool_runtime:
        - provider_id: file-search
          provider_type: inline::file-search
    
    storage:
      backends:
        kv_default:
          type: kv_redis
          url: redis://redis.ogx-storage.svc.cluster.local:6379
        
        sql_default:
          type: sql_postgres
          host: postgresql.ogx-storage.svc.cluster.local
          port: 5432
          db: ogx
          user: ogx_user
          password: \${env.POSTGRES_PASSWORD}
      
      stores:
        metadata:
          namespace: registry
          backend: kv_default
        
        inference:
          table_name: inference_store
          backend: sql_default
          max_write_queue_size: 10000
          num_writers: 4
        
        conversations:
          table_name: conversations
          backend: sql_default
        
        prompts:
          table_name: prompts
          backend: sql_default
    
    server:
      port: 8321
      audit:
        enabled: true
        backend: sql_default
        log_level: info
        include_request_body: false  # Don't log prompts (PHI/PII)
        include_response_body: false  # Don't log responses (PHI/PII)
        fields:
          - timestamp
          - principal
          - model
          - provider
          - duration_ms
          - status_code
          - input_tokens
          - output_tokens
    
    registered_resources:
      models:
        # ⚠️ vLLM model - REMOVE this if you skipped vLLM deployment
        - model_id: llama-3.3-70b
          provider_id: vllm
          model_type: llm
        
        # ✅ Ollama model - CPU-only
        - model_id: llama-3.2-3b
          provider_id: ollama
          model_type: llm
    
    # Multi-tenancy: Per-researcher access control
    access_control:
      # Researcher A: Medical imaging research
      - principal: "researcher:researcher-a"
        allowed_models:
          - "llama-3.3-70b"  # ⚠️ vLLM (GPU) - remove if skipped vLLM deployment
          - "llama-3.2-3b"   # ✅ Ollama (CPU-only)
        allowed_tools:
          - "file_search"
        quota:
          requests_per_day: 1000
          requests_per_hour: 100
          max_tokens_per_request: 8000
          max_concurrent_requests: 5
      
      # Researcher B: Clinical NLP
      - principal: "researcher:researcher-b"
        allowed_models:
          - "llama-3.3-70b"  # ⚠️ vLLM (GPU) - remove if skipped vLLM deployment
        allowed_tools:
          - "file_search"
        quota:
          requests_per_day: 5000
          requests_per_hour: 500
          max_tokens_per_request: 4000
          max_concurrent_requests: 10
      
      # Default: New researchers (conservative limits, CPU-only for testing)
      - principal: "researcher:default"
        allowed_models:
          - "llama-3.2-3b"  # ✅ Ollama (CPU-only)
        allowed_tools:
          - "file_search"
        quota:
          requests_per_day: 100
          requests_per_hour: 20
          max_tokens_per_request: 2000
          max_concurrent_requests: 2
EOF
```

**Deploy OGX via Operator:**

```bash
cat <<EOF | oc apply -f -
---
apiVersion: ogx.io/v1alpha1
kind: OGXDistribution
metadata:
  name: ogx-barcelona
  namespace: ogx-platform
spec:
  replicas: 3
  server:
    distribution:
      name: starter  # Use the starter distribution
    containerSpec:
      port: 8321
      env:
      # PostgreSQL credentials
      - name: POSTGRES_PASSWORD
        valueFrom:
          secretKeyRef:
            name: postgresql-credentials
            key: password
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "4Gi"
          cpu: "2000m"
    # Override default config with our multi-tenant setup
    userConfig:
      configMap:
        name: ogx-config
    storage:
      size: "50Gi"
      storageClassName: ocs-external-storagecluster-ceph-rbd
      mountPath: "/data/ogx"
EOF

# Wait for the operator to create the deployment
sleep 10

# Wait for OGX pods to be ready
oc wait --for=condition=ready pod -l app.kubernetes.io/name=ogx,app.kubernetes.io/instance=ogx-barcelona -n ogx-platform --timeout=300s

# Check the OGXDistribution status
oc get ogxdistribution ogx-barcelona -n ogx-platform
oc describe ogxdistribution ogx-barcelona -n ogx-platform
```

**Create OpenShift Route:**

The operator creates a Service but not a Route. Create the Route manually:

```bash
cat <<EOF | oc apply -f -
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ogx-api
  namespace: ogx-platform
spec:
  host: ogx-api.apps.barcelona.nerc.mghpcc.org
  to:
    kind: Service
    name: ogx-barcelona-service  # Service created by operator
  port:
    targetPort: 8321
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
EOF
```

**Test OGX Health:**

```bash
# Internal health check
oc exec -n ogx-platform -l app.kubernetes.io/instance=ogx-barcelona -- curl -s localhost:8321/health

# External health check
curl -k https://ogx-api.apps.barcelona.nerc.mghpcc.org/health
# Should return: {"status": "healthy"}
```

### Step 5: Verify Deployment

```bash
# Check OGXDistribution status
oc get ogxdistribution ogx-barcelona -n ogx-platform
# Should show: READY=True, REPLICAS=3/3

# Check all pods are running
oc get pods -n ogx-platform -l app.kubernetes.io/instance=ogx-barcelona
oc get pods -n ogx-inference
oc get pods -n ogx-storage

# Verify operator-created resources
oc get deployment,service,pvc -n ogx-platform -l app.kubernetes.io/instance=ogx-barcelona

# Test OGX health
curl -k https://ogx-api.apps.barcelona.nerc.mghpcc.org/health
# Should return: {"status": "healthy"}

# Test inference through OGX (requires API key setup - see Multi-Tenancy section)
curl -k https://ogx-api.apps.barcelona.nerc.mghpcc.org/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer researcher-a-key" \
  -d '{
    "model": "llama-3.3-70b",
    "messages": [{"role": "user", "content": "Hello"}],
    "max_tokens": 50
  }'
```

---

## Multi-Tenancy Setup

### Researcher Namespace Pattern

Each researcher gets:
1. **Dedicated namespace** for their applications and data
2. **NetworkPolicy** that only allows egress to OGX (no cross-researcher traffic)
3. **ResourceQuota** to limit compute resources
4. **RoleBinding** giving them edit access to their namespace
5. **OGX API key** with quotas configured in OGX config

### Creating a New Researcher Namespace

**Script: `create-researcher.sh`**

```bash
#!/bin/bash
# Usage: ./create-researcher.sh researcher-c "Dr. Jane Smith" 2000

RESEARCHER_ID=$1
RESEARCHER_NAME=$2
DAILY_QUOTA=${3:-1000}  # Default 1000 requests/day

# 1. Create namespace
oc new-project ${RESEARCHER_ID} \
  --description="Research workspace for ${RESEARCHER_NAME}"

# 2. Create API key secret
API_KEY="${RESEARCHER_ID}-$(openssl rand -hex 16)"
oc create secret generic ogx-api-key \
  -n ${RESEARCHER_ID} \
  --from-literal=api-key="${API_KEY}"

echo "Created API key: ${API_KEY}"
echo "Store this securely - it won't be shown again!"

# 3. Set resource quotas
cat <<EOF | oc apply -f -
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ${RESEARCHER_ID}-quota
  namespace: ${RESEARCHER_ID}
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "32Gi"
    limits.cpu: "16"
    limits.memory: "64Gi"
    persistentvolumeclaims: "10"
    requests.storage: "100Gi"
EOF

# 4. Create NetworkPolicy (only allow OGX access)
cat <<EOF | oc apply -f -
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ogx-only
  namespace: ${RESEARCHER_ID}
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-dns
    ports:
    - protocol: UDP
      port: 53
  # Allow OGX API
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ogx-platform
    ports:
    - protocol: TCP
      port: 8321
  # Allow external HTTPS (for downloading packages, etc.)
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
EOF

# 5. Grant researcher edit access to their namespace
oc adm policy add-role-to-user edit ${RESEARCHER_NAME} -n ${RESEARCHER_ID}

# 6. Add researcher to OGX access control
# (This requires updating the ogx-config ConfigMap and restarting OGX)
echo "⚠️  MANUAL STEP REQUIRED:"
echo "Add the following to ogx-config ConfigMap access_control section:"
cat <<EOF

  - principal: "researcher:${RESEARCHER_ID}"
    allowed_models:
      - "llama-3.3-70b"
      - "llama-3.2-3b"
    allowed_tools:
      - "file_search"
    quota:
      requests_per_day: ${DAILY_QUOTA}
      requests_per_hour: $(( DAILY_QUOTA / 10 ))
      max_tokens_per_request: 4000
      max_concurrent_requests: 5
EOF

echo ""
echo "Then update the ConfigMap and the operator will restart OGX automatically:"
echo "  1. Edit: oc edit configmap ogx-config -n ogx-platform"
echo "  2. Or manually restart: oc rollout restart deployment/ogx-barcelona -n ogx-platform"
```

**Usage:**

```bash
chmod +x create-researcher.sh
./create-researcher.sh researcher-c "dr.jane.smith@example.edu" 2000
```

---

## Data Isolation & Security

### Network Isolation

**Current setup ensures:**
- ✅ Researchers can only talk to OGX (NetworkPolicy enforced)
- ✅ Researchers cannot access each other's namespaces
- ✅ Researchers cannot directly access inference backends
- ✅ OGX is the only path to AI resources

**Verify isolation:**

```bash
# From researcher-a namespace, try to access researcher-b
oc run test-pod -n researcher-a --image=curlimages/curl --rm -it -- \
  curl http://some-service.researcher-b.svc.cluster.local

# Should timeout (blocked by NetworkPolicy)
```

### Data Segregation in PostgreSQL

**Row-Level Security (RLS):**

```sql
-- Connect to PostgreSQL
oc exec -n ogx-storage -it postgresql-0 -- psql -U ogx_user -d ogx

-- Enable RLS on sensitive tables
ALTER TABLE files_metadata ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE responses ENABLE ROW LEVEL SECURITY;

-- Create policy: users can only see their own data
CREATE POLICY researcher_isolation ON files_metadata
  FOR ALL
  USING (owner = current_user);

CREATE POLICY researcher_isolation ON conversations
  FOR ALL
  USING (owner = current_user);

CREATE POLICY researcher_isolation ON responses
  FOR ALL
  USING (owner = current_user);

-- Create separate database roles per researcher
CREATE ROLE researcher_a WITH LOGIN PASSWORD 'secure_password_a';
CREATE ROLE researcher_b WITH LOGIN PASSWORD 'secure_password_b';

-- Grant only necessary permissions
GRANT SELECT, INSERT, UPDATE ON files_metadata TO researcher_a;
GRANT SELECT, INSERT, UPDATE ON conversations TO researcher_a;
GRANT SELECT, INSERT, UPDATE ON responses TO researcher_a;

-- Repeat for researcher_b, researcher_c, etc.
```

**OGX Connection Pooling per Researcher:**

Modify OGX to use different database credentials per researcher (advanced setup):

```yaml
# ogx-config.yaml (advanced multi-tenant storage)
storage:
  backends:
    sql_researcher_a:
      type: sql_postgres
      host: postgresql.ogx-storage.svc.cluster.local
      user: researcher_a
      password: ${env.RESEARCHER_A_DB_PASSWORD}
    
    sql_researcher_b:
      type: sql_postgres
      host: postgresql.ogx-storage.svc.cluster.local
      user: researcher_b
      password: ${env.RESEARCHER_B_DB_PASSWORD}
```

### Encryption

**In Transit:**
- ✅ TLS on OGX Route (edge termination)
- ✅ Internal traffic can use Service Mesh for mTLS (optional)

**At Rest:**
- ✅ Ceph RBD supports encryption at rest (check with cluster admin)
- ✅ PostgreSQL can use pgcrypto for column-level encryption
- ⚠️ Model weights stored unencrypted (typically acceptable)

**Enable PostgreSQL encryption for sensitive fields:**

```sql
-- Install pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Encrypt sensitive data
CREATE TABLE patient_data (
  id SERIAL PRIMARY KEY,
  researcher_id TEXT NOT NULL,
  encrypted_data BYTEA,  -- Encrypted with pgp_sym_encrypt
  created_at TIMESTAMP DEFAULT NOW()
);

-- Insert encrypted data
INSERT INTO patient_data (researcher_id, encrypted_data)
VALUES ('researcher-a', pgp_sym_encrypt('sensitive patient info', 'encryption_key'));

-- Query encrypted data
SELECT researcher_id, pgp_sym_decrypt(encrypted_data, 'encryption_key') AS data
FROM patient_data
WHERE researcher_id = 'researcher-a';
```

---

## Usage Tracking Configuration

### Built-In OGX Audit Logging

OGX logs every request to PostgreSQL `inference_store` and `audit_log` tables.

**Query usage by researcher:**

```sql
-- Connect to database
oc exec -n ogx-storage -it postgresql-0 -- psql -U ogx_user -d ogx

-- Daily usage per researcher
SELECT
  principal,
  DATE(timestamp) as date,
  COUNT(*) as requests,
  SUM(input_tokens) as input_tokens,
  SUM(output_tokens) as output_tokens,
  AVG(duration_ms) as avg_latency_ms
FROM inference_store
WHERE timestamp >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY principal, DATE(timestamp)
ORDER BY date DESC, principal;

-- Model usage breakdown
SELECT
  principal,
  model,
  COUNT(*) as requests,
  SUM(input_tokens + output_tokens) as total_tokens
FROM inference_store
WHERE timestamp >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY principal, model
ORDER BY principal, total_tokens DESC;

-- Quota utilization (current day)
SELECT
  principal,
  COUNT(*) as requests_today,
  MAX(quota_limit) as daily_limit,
  (COUNT(*)::FLOAT / MAX(quota_limit)) * 100 as percent_used
FROM inference_store
WHERE timestamp >= CURRENT_DATE
GROUP BY principal;
```

### Export Usage Reports

**Monthly report script:**

```bash
#!/bin/bash
# monthly-usage-report.sh

MONTH=$(date +%Y-%m)

oc exec -n ogx-storage postgresql-0 -- psql -U ogx_user -d ogx -c "
COPY (
  SELECT
    principal,
    DATE(timestamp) as date,
    COUNT(*) as requests,
    SUM(input_tokens) as input_tokens,
    SUM(output_tokens) as output_tokens,
    SUM(input_tokens + output_tokens) as total_tokens
  FROM inference_store
  WHERE DATE_TRUNC('month', timestamp) = '${MONTH}-01'::DATE
  GROUP BY principal, DATE(timestamp)
  ORDER BY principal, date
) TO STDOUT WITH CSV HEADER
" > usage_report_${MONTH}.csv

echo "Report generated: usage_report_${MONTH}.csv"
```

**Usage:**

```bash
chmod +x monthly-usage-report.sh
./monthly-usage-report.sh
```

### Real-Time Usage Dashboard (Grafana)

Barcelona already has Grafana Operator installed. Create a dashboard:

```bash
# Create Grafana dashboard ConfigMap
cat <<EOF | oc apply -f -
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ogx-usage-dashboard
  namespace: grafana-operator
  labels:
    grafana_dashboard: "true"
data:
  ogx-usage.json: |
    {
      "dashboard": {
        "title": "OGX Usage Tracking",
        "panels": [
          {
            "title": "Requests per Researcher (24h)",
            "targets": [{
              "expr": "sum by (principal) (rate(ogx_requests_total[24h]))"
            }]
          },
          {
            "title": "Token Usage by Researcher",
            "targets": [{
              "expr": "sum by (principal) (ogx_tokens_total)"
            }]
          },
          {
            "title": "Quota Utilization",
            "targets": [{
              "expr": "ogx_quota_used / ogx_quota_limit"
            }]
          }
        ]
      }
    }
EOF
```

---

## HIPAA/Healthcare Compliance

### Compliance Requirements

For healthcare/medical data (HIPAA, HITECH):

1. **Encryption at Rest** - ✅ Ceph storage (verify with admin)
2. **Encryption in Transit** - ✅ TLS on Routes
3. **Access Control** - ✅ NetworkPolicies + OGX quotas
4. **Audit Logging** - ✅ OGX audit logs every request
5. **Data Segregation** - ✅ Per-researcher namespaces + RLS
6. **No PHI in Logs** - ✅ OGX configured to NOT log request/response bodies

### Audit Trail Requirements

**OGX audit configuration (already in config above):**

```yaml
server:
  audit:
    enabled: true
    include_request_body: false  # ✅ Don't log prompts (may contain PHI)
    include_response_body: false  # ✅ Don't log responses (may contain PHI)
```

**What IS logged:**
- Timestamp
- Researcher identity (principal)
- Model used
- Token counts
- Request duration
- Success/failure status

**What is NOT logged:**
- Actual prompts (may contain patient data)
- Actual responses (may contain PHI)

### Data Retention Policy

**Implement retention policy:**

```sql
-- Create cron job to delete old audit logs (> 7 years for HIPAA)
-- Run this as a PostgreSQL cron job or Kubernetes CronJob

DELETE FROM audit_log
WHERE timestamp < NOW() - INTERVAL '7 years';

DELETE FROM inference_store
WHERE timestamp < NOW() - INTERVAL '7 years';
```

**Kubernetes CronJob for cleanup:**

```bash
cat <<EOF | oc apply -f -
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: audit-log-cleanup
  namespace: ogx-storage
spec:
  schedule: "0 2 * * 0"  # Weekly, Sunday at 2am
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: postgres:16
            command:
            - /bin/bash
            - -c
            - |
              PGPASSWORD=\$POSTGRES_PASSWORD psql -h postgresql -U ogx_user -d ogx <<SQL
              DELETE FROM audit_log WHERE timestamp < NOW() - INTERVAL '7 years';
              DELETE FROM inference_store WHERE timestamp < NOW() - INTERVAL '7 years';
              VACUUM ANALYZE audit_log;
              VACUUM ANALYZE inference_store;
              SQL
            env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgresql-credentials
                  key: password
          restartPolicy: OnFailure
EOF
```

### Business Associate Agreement (BAA)

**Important:** OGX is self-hosted, so no BAA with third-party vendor needed. However:

1. **Document that all PHI stays on barcelona cluster**
2. **No data sent to external APIs** (OpenAI, Anthropic, etc.)
3. **All models run locally** (vLLM on H100s, Ollama on CPU)
4. **Access controls documented** (NetworkPolicies, quotas)
5. **Audit trail preserved** (7+ years retention)

---

## GPU Scheduling

### Current GPU Setup

Barcelona has **20 H100-80GB GPUs** across 5 nodes (4 per node):

- moc-r4pcc02u15-yunshi: 4x H100-80GB
- moc-r4pcc02u16-yunshi: 4x H100-80GB
- moc-r4pcc02u17-nairr: 4x H100-80GB
- moc-r4pcc02u18-nairr: 4x H100-80GB
- moc-r4pcc02u25-nairr: 4x H100-80GB

**GPU Detection:**
```bash
# All nodes have these labels
nvidia.com/gpu.product: NVIDIA-H100-80GB-HBM3
nvidia.com/gpu.count: 4
nvidia.com/gpu.memory: 81559  # MB per GPU
```

**GPU Resource:**
```
nvidia.com/gpu: 4  (per node)
```

### vLLM GPU Scheduling

**Current vLLM deployment uses 4 GPUs** (one node):

```yaml
# No specific nodeSelector needed - scheduler picks any GPU node
resources:
  limits:
    nvidia.com/gpu: "4"  # All 4 GPUs on one node
```

**This leaves 16 H100 GPUs available** across 4 remaining nodes for:
- Additional vLLM instances (different models)
- Multiple concurrent model serving
- Jupyter notebooks for researchers
- Training workloads
- Fine-tuning experiments
- Other GPU tasks

### Scaling vLLM Replicas

**If you need higher throughput, you can scale across all 5 GPU nodes:**

```yaml
# Scale vLLM to use all 5 H100 nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
spec:
  replicas: 5  # One pod per H100 node
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: vllm
            topologyKey: kubernetes.io/hostname
      resources:
        limits:
          nvidia.com/gpu: "4"  # Each pod uses all 4 GPUs on its node
      # ... rest of spec
```

This configuration:
- Deploys 1 vLLM pod per GPU node (5 total)
- Uses all 20 H100 GPUs for maximum throughput
- Provides 5x the inference capacity
- Offers redundancy (if one node fails, 4 remain)
- Enables load balancing across replicas via OGX

### Researcher GPU Access

**If researchers need direct GPU access** (e.g., for Jupyter notebooks):

```yaml
# researcher-a can request GPUs in their namespace
apiVersion: v1
kind: Pod
metadata:
  name: jupyter-notebook
  namespace: researcher-a
spec:
  nodeSelector:
    nvidia.com/gpu.product: NVIDIA-H100-80GB-HBM3
  containers:
  - name: jupyter
    image: jupyter/pytorch-notebook:latest
    resources:
      limits:
        nvidia.com/gpu: "1"  # Request 1 GPU (out of 4 on the node)
```

**ResourceQuota to limit GPU usage:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: researcher-a-gpu-quota
  namespace: researcher-a
spec:
  hard:
    requests.nvidia.com/gpu: "2"  # Max 2 GPUs total
```

---

## Monitoring & Operations

### Health Checks

**Daily health check script:**

```bash
#!/bin/bash
# health-check.sh

echo "=== OGX Health Check ==="
echo ""

echo "1. OGX API Health:"
curl -s https://ogx-api.apps.barcelona.nerc.mghpcc.org/health | jq .

echo ""
echo "2. OGX Pods:"
oc get pods -n ogx-platform

echo ""
echo "3. Inference Backends:"
oc get pods -n ogx-inference

echo ""
echo "4. Storage:"
oc get pods -n ogx-storage

echo ""
echo "5. vLLM Health:"
oc exec -n ogx-inference deployment/vllm -- curl -s localhost:8000/health | jq .

echo ""
echo "6. PostgreSQL Connections:"
oc exec -n ogx-storage postgresql-0 -- psql -U ogx_user -d ogx -c \
  "SELECT count(*) as active_connections FROM pg_stat_activity WHERE state = 'active';"

echo ""
echo "7. Disk Usage:"
oc exec -n ogx-storage postgresql-0 -- df -h /var/lib/postgresql/data
```

### Monitoring with Prometheus

Barcelona already has Prometheus. Create ServiceMonitor for OGX:

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ogx-metrics
  namespace: ogx-platform
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ogx
      app.kubernetes.io/instance: ogx-barcelona
  endpoints:
  - port: 8321  # OGX server port
    interval: 30s
    path: /metrics
```

### Backup Strategy

**PostgreSQL Backups:**

```bash
# Daily backup CronJob
cat <<EOF | oc apply -f -
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgresql-backup
  namespace: ogx-storage
spec:
  schedule: "0 2 * * *"  # Daily at 2am
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:16
            command:
            - /bin/bash
            - -c
            - |
              BACKUP_FILE="/backups/ogx-\$(date +%Y%m%d-%H%M%S).sql"
              PGPASSWORD=\$POSTGRES_PASSWORD pg_dump \
                -h postgresql \
                -U ogx_user \
                -d ogx \
                -F c \
                -f \$BACKUP_FILE
              echo "Backup created: \$BACKUP_FILE"
              
              # Keep only last 30 days
              find /backups -name "ogx-*.sql" -mtime +30 -delete
            env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgresql-credentials
                  key: password
            volumeMounts:
            - name: backups
              mountPath: /backups
          volumes:
          - name: backups
            persistentVolumeClaim:
              claimName: postgresql-backups
          restartPolicy: OnFailure
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-backups
  namespace: ogx-storage
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ocs-external-storagecluster-ceph-rbd
  resources:
    requests:
      storage: 200Gi
EOF
```

---

## Researcher Onboarding

### Onboarding Checklist

When a new researcher joins:

1. **Create namespace** (use `create-researcher.sh` script)
2. **Issue API key** (stored in secret in researcher namespace)
3. **Configure OGX quotas** (update ogx-config ConfigMap)
4. **Send onboarding email** (template below)
5. **Schedule training session** (how to use OGX, best practices)

### Onboarding Email Template

```
Subject: OGX Access - Barcelona Research Cluster

Hi [Researcher Name],

Welcome to the Barcelona research cluster! You now have access to OGX, our multi-tenant AI platform.

**Your Details:**
- Namespace: [researcher-id]
- API Key: [api-key] (keep this secure!)
- Daily Quota: [quota] requests/day
- Allowed Models: llama-3.3-70b, llama-3.2-3b

**Getting Started:**

1. Install the OpenAI Python SDK:
   pip install openai

2. Use OGX in your code:
   
   from openai import OpenAI
   
   client = OpenAI(
       base_url="https://ogx-api.apps.barcelona.nerc.mghpcc.org/v1",
       api_key="[your-api-key]"
   )
   
   response = client.chat.completions.create(
       model="llama-3.3-70b",
       messages=[{"role": "user", "content": "Hello"}]
   )

3. Deploy your applications in your namespace ([researcher-id])

**Important:**
- Your data is isolated from other researchers
- Do NOT log prompts/responses containing PHI
- All requests are audited for compliance
- Review usage at: [grafana-dashboard-url]

**Documentation:**
- OGX Guide: https://ogx-ai.github.io/docs
- Barcelona Cluster Docs: [internal-wiki-url]

Questions? Contact: [support-email]
```

---

## Troubleshooting

### Issue: OGX Pods Not Starting

**Symptom:**
```bash
oc get pods -n ogx-platform -l app.kubernetes.io/instance=ogx-barcelona
# NAME                            READY   STATUS             RESTARTS
# ogx-barcelona-xxxxx             0/1     CrashLoopBackOff   5
```

**Diagnosis:**
```bash
# Check OGXDistribution status
oc describe ogxdistribution ogx-barcelona -n ogx-platform

# Check pod logs
oc logs -n ogx-platform -l app.kubernetes.io/instance=ogx-barcelona

# Common causes:
# - Database connection failure
# - Invalid config
# - Missing secrets
# - Operator issues
```

**Solution:**
```bash
# 1. Verify PostgreSQL is running
oc get pods -n ogx-storage -l app=postgresql

# 2. Test database connectivity from OGX pod
POD=$(oc get pod -n ogx-platform -l app.kubernetes.io/instance=ogx-barcelona -o jsonpath='{.items[0].metadata.name}')
oc exec -n ogx-platform $POD -- \
  nc -zv postgresql.ogx-storage.svc.cluster.local 5432

# 3. Check config
oc get configmap ogx-config -n ogx-platform -o yaml

# 4. Check secrets
oc get secret postgresql-credentials -n ogx-storage

# 5. Check operator logs if issue persists
oc logs -n ogx-k8s-operator-system -l control-plane=controller-manager
```

### Issue: vLLM Out of Memory

**Symptom:**
```bash
oc logs -n ogx-inference deployment/vllm
# CUDA out of memory
```

**Solution:**
```bash
# Reduce model size or max_model_len
oc set env deployment/vllm \
  -n ogx-inference \
  MAX_MODEL_LEN=4096  # Reduce from 8192

# Or use smaller model
# Switch to Llama-3.1-8B instead of 70B
```

### Issue: Researcher Hitting Quota

**Symptom:**
Researcher reports `429 Too Many Requests`

**Diagnosis:**
```sql
-- Check current usage
oc exec -n ogx-storage postgresql-0 -- psql -U ogx_user -d ogx -c "
SELECT
  principal,
  COUNT(*) as requests_today
FROM inference_store
WHERE timestamp >= CURRENT_DATE
  AND principal = 'researcher:researcher-a'
GROUP BY principal;
"
```

**Solution:**
```bash
# Option 1: Increase quota (edit ogx-config)
oc edit configmap ogx-config -n ogx-platform
# Change quota.requests_per_day: 5000

# The operator should automatically reload config, or manually restart:
oc rollout restart deployment/ogx-barcelona -n ogx-platform

# Option 2: Reset quota manually (emergency)
POD=$(oc get pod -n ogx-platform -l app.kubernetes.io/instance=ogx-barcelona -o jsonpath='{.items[0].metadata.name}')
oc exec -n ogx-platform $POD -- \
  ogx admin quota-reset --researcher researcher-a
```

### Issue: GPU Not Being Used

**Symptom:**
```bash
# vLLM running but not using GPU
oc exec -n ogx-inference deployment/vllm -- nvidia-smi
# No processes using GPU
```

**Diagnosis:**
```bash
# Check GPU allocation
oc describe pod -n ogx-inference -l app=vllm | grep nvidia.com/gpu

# Check GPU Operator
oc get pods -n nvidia-gpu-operator
```

**Solution:**
```bash
# 1. Verify GPU request in deployment
oc get deployment vllm -n ogx-inference -o yaml | grep -A 3 resources

# Should see:
#   resources:
#     limits:
#       nvidia.com/gpu: "4"

# 2. Check node labels
oc get nodes --show-labels | grep h100

# 3. Restart pod
oc delete pod -n ogx-inference -l app=vllm
```

---

## Next Steps

### Immediate (Week 1):
- [ ] Deploy storage layer (PostgreSQL, Redis)
- [ ] Download Llama model (takes ~2 hours)
- [ ] Deploy vLLM with H100 GPUs
- [ ] Deploy Ollama (CPU fallback)
- [ ] Deploy OGX platform
- [ ] Test with simple inference request

### Short-term (Week 2-4):
- [ ] Create researcher namespaces (researcher-c, researcher-d, etc.)
- [ ] Configure per-researcher quotas
- [ ] Set up Grafana dashboards for usage tracking
- [ ] Implement PostgreSQL backup CronJob
- [ ] Document onboarding process
- [ ] Train first researchers on OGX usage

### Medium-term (Month 2-3):
- [ ] Enable Row-Level Security in PostgreSQL
- [ ] Set up automated usage reports
- [ ] Implement data retention policy (7-year HIPAA)
- [ ] Add more models (Mistral, CodeLlama, etc.)
- [ ] Scale vLLM to use both H100 nodes
- [ ] Set up monitoring alerts (PagerDuty, Slack)

### Long-term (Quarter 2+):
- [ ] Integrate with institutional SSO (LDAP/OIDC)
- [ ] Implement MCP server integration for tool calling
- [ ] Add vector store (pgvector) for RAG workloads
- [ ] Deploy model fine-tuning pipeline
- [ ] Formal HIPAA compliance audit
- [ ] Publish internal best practices guide

---

## Appendix: Useful Commands

### Quick Reference

```bash
# Check cluster status
oc get nodes
oc get pods --all-namespaces

# OGX logs
oc logs -f -n ogx-platform -l app.kubernetes.io/instance=ogx-barcelona

# vLLM logs
oc logs -f -n ogx-inference deployment/vllm

# Database shell
oc exec -n ogx-storage -it postgresql-0 -- psql -U ogx_user -d ogx

# Test OGX health
curl -k https://ogx-api.apps.barcelona.nerc.mghpcc.org/health

# View researcher quotas
oc get configmap ogx-config -n ogx-platform -o yaml | grep -A 10 access_control

# Check GPU usage
oc exec -n ogx-inference deployment/vllm -- nvidia-smi

# View usage stats
oc exec -n ogx-storage postgresql-0 -- psql -U ogx_user -d ogx -c \
  "SELECT principal, COUNT(*) FROM inference_store WHERE timestamp >= CURRENT_DATE GROUP BY principal;"
```

---

## Support & Resources

**Barcelona Cluster:**
- Cluster Admin: [contact info]
- Documentation: [internal wiki]
- Status Page: [status URL]

**OGX:**
- Documentation: https://ogx-ai.github.io/docs
- GitHub: https://github.com/ogx-ai/ogx
- Discord: https://discord.gg/ZAFjsrcw

**HIPAA Compliance:**
- HHS HIPAA Resources: https://www.hhs.gov/hipaa
- NIST Cybersecurity Framework: https://www.nist.gov/cyberframework

---

**Document Version:** 1.0  
**Last Updated:** 2026-05-28  
**Author:** EldritchJS  
**Cluster:** barcelona.nerc.mghpcc.org
