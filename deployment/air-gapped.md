# Air-Gapped Deployment

## Running Sensei Without Internet Connectivity

Air-gapped deployments are required for organizations with strict security requirements, classified environments, or regulatory constraints that prohibit internet connectivity. Sensei fully supports air-gapped operation with some architectural considerations.

---

## Air-Gapped Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                    AIR-GAPPED ENVIRONMENT                       │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   MABA      │  │   KORA      │  │   AMANI     │             │
│  │  (Local)    │  │  (Local)    │  │  (Local)    │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
│  ┌───────────────────────┴───────────────────────┐             │
│  │              Sensei Core Platform              │             │
│  │  • Local LLMs (Llama-3, Mistral)              │             │
│  │  • Local Vector DB (Qdrant/Milvus)            │             │
│  │  • Local Pattern Library                       │             │
│  └───────────────────────────────────────────────┘             │
│                          │                                      │
│  ┌───────────────────────┴───────────────────────┐             │
│  │              GPU Inference Cluster             │             │
│  │              (NVIDIA A100/H100)                │             │
│  └───────────────────────────────────────────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                           │
                    [Physical Gap]
                           │
┌─────────────────────────────────────────────────────────────────┐
│                    TRANSFER STATION                             │
│  • Update packages (sneakernet)                                 │
│  • Pattern library updates                                      │
│  • License validation (offline tokens)                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Requirements

### Hardware Requirements

| Component          | Minimum            | Recommended         | Purpose                  |
| ------------------ | ------------------ | ------------------- | ------------------------ |
| **Cognitive Tier** | 8x NVIDIA A10      | 4x NVIDIA A100      | Local LLM inference      |
| **Execution Tier** | 16 cores, 64GB RAM | 32 cores, 128GB RAM | Worker agents            |
| **Storage**        | 2TB NVMe           | 10TB NVMe           | Models, cache, artifacts |
| **Network**        | 10GbE internal     | 25GbE internal      | Inter-node communication |

### Software Requirements

| Component             | Version                      | Notes                          |
| --------------------- | ---------------------------- | ------------------------------ |
| **Operating System**  | RHEL 8.6+, Ubuntu 22.04      | Hardened OS images supported   |
| **Container Runtime** | containerd 1.6+, CRI-O 1.24+ | Docker not required            |
| **Kubernetes**        | 1.27+                        | OpenShift 4.12+ also supported |
| **GPU Drivers**       | NVIDIA 525+                  | CUDA 12.0+                     |
| **Python**            | 3.11+                        | For local model serving        |

---

## Local LLM Configuration

Air-gapped deployments use locally-hosted language models instead of cloud APIs.

### Supported Local Models

| Model             | Size  | VRAM Required  | Quality   | Speed     |
| ----------------- | ----- | -------------- | --------- | --------- |
| **Llama-3-70B**   | 70B   | 140GB (A100x2) | Excellent | Moderate  |
| **Llama-3-8B**    | 8B    | 16GB           | Good      | Excellent |
| **Mistral-7B**    | 7B    | 14GB           | Good      | Excellent |
| **CodeLlama-34B** | 34B   | 68GB           | Very Good | Good      |
| **Mixtral-8x7B**  | 46.7B | 100GB          | Very Good | Good      |

### Configuration

```yaml
# Air-gapped LLM configuration
agents:
  cognitive:
    # Use local models instead of cloud APIs
    scout_model: local://llama-3-70b
    architect_model: local://llama-3-70b

    # Local inference settings
    local_inference:
      enabled: true
      endpoint: http://vllm.sensei.internal:8000

      # Model serving configuration
      vllm:
        model_path: /models/llama-3-70b
        tensor_parallel_size: 2 # Number of GPUs
        max_model_len: 32768
        quantization: awq # Optional: awq, gptq, none

  execution:
    worker_model: local://llama-3-8b

    local_inference:
      enabled: true
      endpoint: http://vllm-workers.sensei.internal:8000
```

### vLLM Deployment

```yaml
# vllm-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-inference
  namespace: sensei
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
      containers:
        - name: vllm
          image: sensei-airgap-registry/vllm:0.3.0
          args:
            - --model=/models/llama-3-70b
            - --tensor-parallel-size=2
            - --max-model-len=32768
          resources:
            limits:
              nvidia.com/gpu: 2
          volumeMounts:
            - name: models
              mountPath: /models
      volumes:
        - name: models
          persistentVolumeClaim:
            claimName: model-storage
```

---

## Local Vector Database

Replace cloud vector databases with self-hosted alternatives:

### Qdrant (Recommended)

```yaml
database:
  vector:
    provider: qdrant
    qdrant:
      host: qdrant.sensei.internal
      port: 6333
      grpc_port: 6334
      collection: sensei-patterns

      # Performance tuning
      optimizers:
        memmap_threshold: 20000
        indexing_threshold: 10000
```

### Milvus

```yaml
database:
  vector:
    provider: milvus
    milvus:
      host: milvus.sensei.internal
      port: 19530
      collection: sensei_patterns
```

---

## Offline Pattern Library

Air-gapped installations ship with a pre-populated pattern library:

### Initial Pattern Library

The installation package includes:

- **50,000+ validated migration patterns**
- **10,000+ error-recovery patterns**
- **Schema fingerprints for common systems** (Oracle, SQL Server, PostgreSQL, MySQL, etc.)
- **Transformation templates** for 200+ common scenarios

### Pattern Library Updates

Updates are delivered via physical media (USB, DVD) or secure transfer:

```bash
# Import pattern library update
sensei patterns import --file pattern-update-2026-Q1.bundle

# Verify import integrity
sensei patterns verify --checksum sha256:abc123...

# Apply updates
sensei patterns apply
```

Update frequency: **Quarterly** (or on-demand for Enterprise tier)

---

## Offline Licensing

Air-gapped installations use offline license validation:

### License Activation

```bash
# Generate machine fingerprint
sensei license fingerprint > fingerprint.txt

# Transfer fingerprint to license server (via secure channel)
# Receive license token

# Activate license
sensei license activate --token offline_license_token.jwt
```

### License Renewal

```bash
# Check license status
sensei license status

# Generate renewal request
sensei license renew-request > renewal-request.txt

# After receiving new token
sensei license renew --token new_license_token.jwt
```

License validity: **1 year** (configurable for Enterprise)

---

## Installation

### Prerequisites

1. **Obtain installation bundle:**

   ```text
   sensei-airgap-v1.5.0.tar.gz
   ├── images/           # Container images
   ├── models/           # LLM model weights
   ├── patterns/         # Initial pattern library
   ├── charts/           # Helm charts
   ├── tools/            # Installation utilities
   └── checksums.txt     # SHA256 verification
   ```

2. **Verify bundle integrity:**
   ```bash
   sha256sum -c checksums.txt
   ```

### Installation Steps

```bash
# 1. Load container images
for image in images/*.tar; do
  ctr -n k8s.io images import "$image"
done

# 2. Deploy pattern library storage
kubectl apply -f charts/storage/

# 3. Load model weights
kubectl cp models/ sensei/model-loader:/models/

# 4. Deploy infrastructure components
helm install sensei-infra charts/infrastructure/ \
  --namespace sensei \
  --values values-airgap.yaml

# 5. Deploy Sensei platform
helm install sensei charts/sensei/ \
  --namespace sensei \
  --values values-airgap.yaml

# 6. Activate license
sensei license activate --token $LICENSE_TOKEN

# 7. Verify installation
sensei health check --comprehensive
```

---

## Operational Considerations

### Updates

| Component        | Update Method                | Frequency         |
| ---------------- | ---------------------------- | ----------------- |
| Platform         | Physical media bundle        | Monthly/Quarterly |
| Pattern Library  | Encrypted bundle             | Quarterly         |
| Models           | Physical media               | As needed         |
| Security patches | Critical: expedited delivery | As needed         |

### Monitoring

Air-gapped environments require local monitoring infrastructure:

```yaml
monitoring:
  # Prometheus (local)
  metrics:
    provider: prometheus
    endpoint: http://prometheus.sensei.internal:9090

  # Jaeger (local)
  tracing:
    provider: jaeger
    endpoint: http://jaeger.sensei.internal:14268

  # Grafana dashboards included
  dashboards:
    path: /etc/sensei/dashboards/
```

### Backup & Recovery

```yaml
backup:
  # Local backup storage
  destination: /backup/sensei/

  # Backup schedule
  schedule: '0 2 * * *' # Daily at 2 AM

  # Retention
  retention:
    daily: 7
    weekly: 4
    monthly: 12

  # Components to backup
  include:
    - metadata_database
    - vector_database
    - pattern_library
    - configuration
    - audit_logs
```

---

## Security Hardening

Air-gapped deployments include additional security measures:

### Network Isolation

```yaml
# Network policy - deny all external
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sensei-airgap
  namespace: sensei
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: sensei
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: sensei
```

### Data Classification

```yaml
security:
  classification:
    level: SECRET # or: TOP_SECRET, UNCLASSIFIED

    # Data handling
    data_at_rest: AES-256-GCM
    data_in_transit: TLS-1.3

    # Audit requirements
    audit:
      all_access: true
      retention: 7y
```

---

## Support

Air-gapped customers receive support via:

1. **Secure email** — Encrypted communication for non-urgent issues
2. **Scheduled calls** — Pre-arranged support sessions
3. **On-site support** — For critical issues (Enterprise tier)
4. **Documentation bundle** — Complete offline documentation included

→ [Self-Hosted Deployment](self-hosted.md)
→ [Security Architecture](../../developers/architecture/technology-stack/security-architecture.md)
→ [Configuration Reference](configuration.md)
