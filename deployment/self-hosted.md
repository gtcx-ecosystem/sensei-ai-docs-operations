# Self-Hosted Deployment

## Fully On-Premises Installation

Self-hosted deployment gives you complete control over Sensei infrastructure. Ideal for air-gapped environments, strict compliance requirements, or organizations that prefer managing their own infrastructure.

---

### Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    YOUR DATACENTER / PRIVATE CLOUD                   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Sensei Platform (Complete)                │   │
│  │                                                               │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │  Control Plane                                        │   │   │
│  │  │  Dashboard │ API │ Orchestration │ Pattern Library   │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  │                                                               │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │  Data Plane                                           │   │   │
│  │  │  MABA │ KORA │ AMANI │ Workers │ Agents              │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  │                                                               │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │  Data Stores                                          │   │   │
│  │  │  PostgreSQL │ Redis │ Vector DB │ Object Storage     │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  │                                                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────┐                    ┌─────────────────┐         │
│  │  Source DB      │                    │  Target DB      │         │
│  └─────────────────┘                    └─────────────────┘         │
│                                                                      │
│  No external connectivity required (air-gapped OK)                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Requirements

#### Kubernetes Cluster

| Component      | Minimum    | Recommended |
| -------------- | ---------- | ----------- |
| Nodes          | 3          | 5+          |
| CPU (total)    | 32 cores   | 64+ cores   |
| Memory (total) | 128 GB     | 256+ GB     |
| Storage        | 500 GB SSD | 1+ TB NVMe  |

#### External Dependencies

| Service         | Purpose                | Options                        |
| --------------- | ---------------------- | ------------------------------ |
| PostgreSQL 14+  | Metadata store         | Self-managed or managed        |
| Redis 7+        | Caching, pub/sub       | Self-managed or managed        |
| Object storage  | Checkpoints, artifacts | S3-compatible (MinIO)          |
| Vector database | Pattern library        | Weaviate (bundled) or external |

#### Optional (for local LLM)

| Component | Requirement       |
| --------- | ----------------- |
| GPU       | NVIDIA A100 40GB+ |
| CUDA      | 12.0+             |
| Driver    | 525.60.13+        |

---

### Installation

#### 1. Obtain License

Contact sales@sensei.ai for self-hosted license and container registry credentials.

#### 2. Pull Container Images

```bash
# Login to Sensei registry
docker login registry.sensei.ai

# Pull images (or use provided offline bundle)
docker pull registry.sensei.ai/sensei/control-plane:1.0.0
docker pull registry.sensei.ai/sensei/data-plane:1.0.0
docker pull registry.sensei.ai/sensei/maba:1.0.0
docker pull registry.sensei.ai/sensei/kora:1.0.0
docker pull registry.sensei.ai/sensei/amani:1.0.0
```

#### 3. Deploy Dependencies

```bash
# Using bundled dependencies (recommended)
helm install sensei-deps sensei/dependencies \
  --namespace sensei \
  --create-namespace

# Or connect to existing:
# - PostgreSQL: set postgresql.external.enabled=true
# - Redis: set redis.external.enabled=true
# - Object storage: set storage.external.enabled=true
```

#### 4. Configure License

```yaml
# license.yaml
license:
  key: 'your-license-key-here'
  organization: 'Your Company'
```

#### 5. Deploy Sensei

```bash
helm install sensei sensei/sensei-full \
  --namespace sensei \
  -f license.yaml \
  -f values.yaml
```

**values.yaml:**

```yaml
global:
  domain: sensei.internal.example.com

controlPlane:
  replicas: 2

dataPlane:
  workers:
    replicas: 4
  maba:
    replicas: 2
  kora:
    replicas: 2
  amani:
    replicas: 2

postgresql:
  external:
    enabled: false # Use bundled

redis:
  external:
    enabled: false

storage:
  type: minio # Bundled MinIO

llm:
  # Option 1: Use external API (requires connectivity)
  provider: anthropic
  apiKey: ${ANTHROPIC_API_KEY}

  # Option 2: Local model (air-gapped)
  # provider: local
  # model: llama-3-70b
  # gpu:
  #   enabled: true
  #   count: 2
```

#### 6. Verify Installation

```bash
# Check all pods running
kubectl get pods -n sensei

# Check services
kubectl get svc -n sensei

# Access dashboard
kubectl port-forward svc/sensei-dashboard 8080:80 -n sensei
# Open https://localhost:8080
```

---

### Air-Gapped Installation

For environments with no internet connectivity:

#### 1. Prepare Offline Bundle

```bash
# On connected machine
sensei bundle create --version 1.0.0 --output sensei-bundle.tar.gz

# Bundle includes:
# - All container images
# - Helm charts
# - Pattern library snapshot
# - Local LLM model (optional)
```

#### 2. Transfer to Air-Gapped Environment

Use approved media transfer process.

#### 3. Load Images

```bash
# Load to local registry
tar -xzf sensei-bundle.tar.gz
./load-images.sh --registry registry.internal.example.com
```

#### 4. Install from Local Registry

```yaml
# values-airgapped.yaml
global:
  imageRegistry: registry.internal.example.com

llm:
  provider: local
  model: llama-3-70b

patternLibrary:
  source: local
  snapshotPath: /data/patterns
```

---

### Updates

Self-hosted updates are manual:

```bash
# 1. Download new version
sensei bundle create --version 1.1.0

# 2. Load new images
./load-images.sh

# 3. Update Helm release
helm upgrade sensei sensei/sensei-full \
  --version 1.1.0 \
  -f values.yaml
```

**Recommended:** Update within 90 days of release for security patches.

---

### Support

Self-hosted customers receive:

- Access to documentation
- Email support
- Quarterly architecture reviews (Enterprise)
- Optional: Professional services for installation

→ [Kubernetes](kubernetes.md)
→ [High Availability](high-availability.md)
→ [Backup & Recovery](backup-recovery.md)
