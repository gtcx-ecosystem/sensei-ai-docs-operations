# Infrastructure Requirements

## Compute, Network, and Storage Planning

This page details infrastructure requirements for Hybrid and Self-Hosted deployments. SaaS deployments have no infrastructure requirements — Sensei manages everything.

---

### Compute Requirements

#### Control Plane

| Component    | CPU     | Memory | Replicas | Notes                         |
| ------------ | ------- | ------ | -------- | ----------------------------- |
| API Server   | 2 cores | 4 GB   | 2+       | Stateless, scale horizontally |
| Dashboard    | 1 core  | 2 GB   | 2+       | Stateless                     |
| Orchestrator | 4 cores | 8 GB   | 2+       | Temporal workflows            |
| Scheduler    | 2 cores | 4 GB   | 1        | Single leader                 |

**Total Control Plane:** ~16 cores, 32 GB RAM (minimum HA)

#### Data Plane

| Component | CPU     | Memory | Replicas | Notes                    |
| --------- | ------- | ------ | -------- | ------------------------ |
| MABA      | 8 cores | 16 GB  | 2+       | CPU-intensive transforms |
| KORA      | 4 cores | 8 GB   | 2+       | Validation engine        |
| AMANI     | 2 cores | 4 GB   | 2+       | Conversational interface |
| Workers   | 4 cores | 8 GB   | 4-32     | Scale with workload      |
| Agents    | 2 cores | 4 GB   | 4+       | Scout, Architect, etc.   |

**Total Data Plane (base):** ~40 cores, 80 GB RAM

#### GPU (Optional)

For local LLM inference:

| Model       | GPU Memory | GPU Type | Inference Speed |
| ----------- | ---------- | -------- | --------------- |
| Llama-3-8B  | 16 GB      | A10, L4  | 50 tok/s        |
| Llama-3-70B | 80 GB      | 2×A100   | 20 tok/s        |
| Mistral-7B  | 14 GB      | A10, L4  | 60 tok/s        |

---

### Sizing Guidelines

| Workload   | Concurrent Migrations | Nodes | Total CPU  | Total RAM |
| ---------- | --------------------- | ----- | ---------- | --------- |
| Small      | 1-2                   | 3     | 24 cores   | 64 GB     |
| Medium     | 3-10                  | 5     | 64 cores   | 160 GB    |
| Large      | 10-50                 | 10    | 160 cores  | 400 GB    |
| Enterprise | 50+                   | 20+   | 400+ cores | 1+ TB     |

---

### Network Requirements

#### Bandwidth

| Traffic Type               | Bandwidth | Notes                   |
| -------------------------- | --------- | ----------------------- |
| Control plane internal     | 100 Mbps  | API, orchestration      |
| Data plane internal        | 1 Gbps    | Worker communication    |
| Source database            | 1 Gbps+   | Scales with data volume |
| Target database            | 1 Gbps+   | Scales with data volume |
| External API (SaaS/Hybrid) | 100 Mbps  | Control traffic only    |

#### Latency

| Connection           | Maximum Latency | Impact              |
| -------------------- | --------------- | ------------------- |
| Worker ↔ Redis       | <5 ms           | Swarm communication |
| Worker ↔ Source DB   | <50 ms          | Read performance    |
| Worker ↔ Target DB   | <50 ms          | Write performance   |
| Control ↔ Data plane | <100 ms         | Orchestration       |

#### DNS

- Internal DNS resolution required
- External DNS for Hybrid/SaaS connectivity
- Recommend dedicated DNS entries for Sensei services

---

### Storage Requirements

#### Persistent Storage

| Store          | Type    | Size    | IOPS  | Notes                            |
| -------------- | ------- | ------- | ----- | -------------------------------- |
| PostgreSQL     | SSD     | 100 GB+ | 3000+ | Metadata, scales with migrations |
| Redis          | SSD     | 20 GB   | 3000+ | Cache, pub/sub                   |
| Vector DB      | SSD     | 50 GB+  | 1000+ | Pattern library                  |
| Object Storage | HDD/SSD | 500 GB+ | N/A   | Checkpoints, artifacts           |

#### Ephemeral Storage

Workers need local scratch space:

| Component | Local Storage | Purpose                |
| --------- | ------------- | ---------------------- |
| Workers   | 50 GB         | Temp files, sorting    |
| MABA      | 100 GB        | Transformation staging |
| KORA      | 50 GB         | Validation artifacts   |

---

### Storage Classes

Kubernetes storage class recommendations:

```yaml
# Fast storage for databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sensei-fast
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"

# Standard storage for objects
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sensei-standard
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
```

---

### Cloud Provider Sizing

#### AWS

| Workload | Node Type   | Count | EBS        | Total Cost (est.) |
| -------- | ----------- | ----- | ---------- | ----------------- |
| Small    | m6i.2xlarge | 3     | 500 GB gp3 | ~$1,200/mo        |
| Medium   | m6i.4xlarge | 5     | 1 TB gp3   | ~$4,000/mo        |
| Large    | m6i.8xlarge | 10    | 2 TB gp3   | ~$15,000/mo       |

#### GCP

| Workload | Node Type      | Count | Disk       | Total Cost (est.) |
| -------- | -------------- | ----- | ---------- | ----------------- |
| Small    | n2-standard-8  | 3     | 500 GB SSD | ~$1,100/mo        |
| Medium   | n2-standard-16 | 5     | 1 TB SSD   | ~$3,800/mo        |
| Large    | n2-standard-32 | 10    | 2 TB SSD   | ~$14,000/mo       |

#### Azure

| Workload | Node Type        | Count | Disk       | Total Cost (est.) |
| -------- | ---------------- | ----- | ---------- | ----------------- |
| Small    | Standard_D8s_v4  | 3     | 500 GB P30 | ~$1,300/mo        |
| Medium   | Standard_D16s_v4 | 5     | 1 TB P30   | ~$4,200/mo        |
| Large    | Standard_D32s_v4 | 10    | 2 TB P30   | ~$16,000/mo       |

---

### Capacity Planning

Plan for peak usage:

```text
Peak Workers = (Peak Concurrent Migrations) × (Workers per Migration)

Example:
- Peak migrations: 10
- Workers per migration: 8
- Peak workers: 80

Infrastructure = Base + (Peak Workers × Resources per Worker)
- Base: 40 cores, 80 GB
- Workers: 80 × 4 cores = 320 cores
- Workers: 80 × 8 GB = 640 GB
- Total: 360 cores, 720 GB RAM
```

Add 20% buffer for overhead and bursts.

→ [Network Configuration](network.md)
→ [Kubernetes](kubernetes.md)
→ [Scaling](scaling.md)
