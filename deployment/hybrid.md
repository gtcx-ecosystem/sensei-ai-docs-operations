# Hybrid Deployment

## Data Plane in Your VPC, Control Plane in Sensei Cloud

Hybrid deployment keeps your data within your network while leveraging Sensei's managed control plane for orchestration, updates, and the web interface.

---

### Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                       SENSEI CLOUD (Control Plane)                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Dashboard │ API │ Orchestration │ Pattern Library          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ Control traffic only
                               │ (HTTPS, no data)
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         YOUR VPC (Data Plane)                        │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Sensei Data Plane                          │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │   │
│  │  │  MABA   │  │  KORA   │  │ Workers │  │  Cache  │        │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│           │                                      │                   │
│           ↓                                      ↓                   │
│  ┌─────────────────┐                    ┌─────────────────┐         │
│  │  Source DB      │                    │  Target DB      │         │
│  └─────────────────┘                    └─────────────────┘         │
│                                                                      │
│  Data never leaves your VPC                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

### What Stays Where

| Component              | Location     | Contains            |
| ---------------------- | ------------ | ------------------- |
| **Dashboard**          | Sensei Cloud | UI, no data         |
| **API Gateway**        | Sensei Cloud | Routing, auth       |
| **Orchestration**      | Sensei Cloud | Job scheduling      |
| **Pattern Library**    | Sensei Cloud | Anonymized patterns |
| **MABA Engine**        | Your VPC     | Data transformation |
| **KORA Validation**    | Your VPC     | Data verification   |
| **Worker Agents**      | Your VPC     | Data processing     |
| **Source/Target Data** | Your VPC     | Never leaves        |

---

### Deployment Steps

#### 1. Prerequisites

- Kubernetes cluster (EKS, GKE, AKS, or self-managed)
- Helm 3.x installed
- Network connectivity to Sensei Cloud
- Sensei account with Hybrid plan

#### 2. Get Deployment Credentials

```bash
# Log into Sensei CLI
sensei auth login

# Get data plane credentials
sensei data-plane credentials > credentials.yaml
```

#### 3. Deploy Data Plane

```bash
# Add Sensei Helm repo
helm repo add sensei https://charts.sensei.ai

# Install data plane
helm install sensei-data-plane sensei/data-plane \
  --namespace sensei \
  --create-namespace \
  -f credentials.yaml \
  -f values.yaml
```

**values.yaml:**

```yaml
dataPlane:
  region: us-east-1

controlPlane:
  endpoint: https://api.sensei.ai

resources:
  workers:
    replicas: 4
    memory: 8Gi
    cpu: 4

  maba:
    replicas: 2
    memory: 16Gi
    cpu: 8

  kora:
    replicas: 2
    memory: 8Gi
    cpu: 4

networking:
  # Your VPC configuration
  vpc:
    id: vpc-abc123
    subnets:
      - subnet-123
      - subnet-456
```

#### 4. Verify Connection

```bash
# Check data plane status
sensei data-plane status

# Expected output:
# Data Plane: connected
# Control Plane: api.sensei.ai
# Workers: 4/4 healthy
# MABA: 2/2 healthy
# KORA: 2/2 healthy
```

#### 5. Configure Database Access

Databases connect directly to data plane (no internet required):

```yaml
source:
  type: postgresql
  host: source-db.internal.example.com # Internal DNS
  port: 5432
```

---

### Network Requirements

| Traffic           | Direction           | Destination         | Port    |
| ----------------- | ------------------- | ------------------- | ------- |
| Control plane API | Data plane → Sensei | api.sensei.ai       | 443     |
| Metrics/logs      | Data plane → Sensei | telemetry.sensei.ai | 443     |
| Pattern library   | Data plane → Sensei | patterns.sensei.ai  | 443     |
| Source database   | Data plane → Source | Your config         | DB port |
| Target database   | Data plane → Target | Your config         | DB port |

**No inbound connections required** — data plane initiates all connections.

---

### Security

| Control         | Implementation                                |
| --------------- | --------------------------------------------- |
| Data isolation  | Data never leaves your VPC                    |
| Control traffic | HTTPS only, mutual TLS available              |
| Credentials     | Stored in your secret manager                 |
| Network         | Private subnets, no public IPs required       |
| Updates         | Control plane auto-updates, data plane manual |

---

### Updates

**Control plane:** Updated automatically by Sensei (no action required)

**Data plane:** Updated via Helm:

```bash
# Check for updates
helm repo update
helm search repo sensei/data-plane --versions

# Update data plane
helm upgrade sensei-data-plane sensei/data-plane \
  --namespace sensei \
  -f credentials.yaml \
  -f values.yaml
```

Recommend updating data plane within 30 days of new release for compatibility.

---

### Scaling

Data plane scales independently:

```bash
# Scale workers
kubectl scale deployment sensei-workers -n sensei --replicas=8

# Or via Helm
helm upgrade sensei-data-plane sensei/data-plane \
  --set resources.workers.replicas=8
```

Auto-scaling available:

```yaml
autoscaling:
  enabled: true
  minReplicas: 4
  maxReplicas: 32
  targetCPUUtilization: 70
```

→ [Self-Hosted Deployment](self-hosted.md)
→ [Kubernetes](kubernetes.md)
→ [Network Configuration](network.md)
