# Resource Management

## Compute, Memory, and Storage

Understanding resource requirements helps you plan infrastructure, estimate costs, and troubleshoot performance issues.

---

### Resource Categories

| Resource    | Used By                                | Scales With             |
| ----------- | -------------------------------------- | ----------------------- |
| **CPU**     | Transformations, validation, agents    | Complexity, parallelism |
| **Memory**  | Batch buffering, LLM inference, caches | Batch size, model size  |
| **Network** | Data transfer, agent communication     | Data volume, geography  |
| **Storage** | Checkpoints, logs, temporary files     | Data volume, retention  |
| **GPU**     | Local LLM inference (optional)         | Model size, throughput  |

---

### CPU Requirements

| Component         | Base    | Per Migration | Notes                   |
| ----------------- | ------- | ------------- | ----------------------- |
| API Server        | 2 cores | +0.5 cores    | Relatively constant     |
| Agent Coordinator | 2 cores | +1 core       | Scales with complexity  |
| Worker Agents     | 4 cores | +2-8 cores    | Main scaling dimension  |
| KORA Validation   | 2 cores | +2-4 cores    | During validation phase |

**Typical deployments:**

- Small (2 concurrent migrations): 16 cores
- Medium (10 concurrent migrations): 64 cores
- Large (50+ concurrent migrations): 256+ cores

---

### Memory Requirements

| Component       | Base    | Per Migration     | Notes                   |
| --------------- | ------- | ----------------- | ----------------------- |
| Agent Framework | 4 GB    | +1 GB             | LangChain, Ray overhead |
| Worker Buffers  | -       | 100 MB per worker | Batch processing        |
| LLM Inference   | 8-16 GB | Shared            | Model stays loaded      |
| Pattern Cache   | 2 GB    | +100 MB           | Growing with patterns   |
| KORA            | 4 GB    | +2 GB             | During validation       |

**Memory sizing:**

```yaml
# Small deployment
resources:
  memory:
    api: 4Gi
    workers: 16Gi      # 16 workers × 1GB each
    llm_inference: 16Gi  # Shared model
    kora: 8Gi

# Large deployment
resources:
  memory:
    api: 8Gi
    workers: 128Gi     # 64 workers × 2GB each
    llm_inference: 32Gi
    kora: 32Gi
```

---

### Network Bandwidth

Data transfer dominates network usage:

| Phase               | Direction                | Volume                  |
| ------------------- | ------------------------ | ----------------------- |
| Schema Analysis     | Source → Sensei          | ~1% of data (sampling)  |
| Execution           | Source → Sensei → Target | 100% of data            |
| Validation          | Target → Sensei          | ~10% of data (sampling) |
| Agent Communication | Internal                 | <1 GB/hr                |

**Bandwidth requirements:**

```text
Data Volume    Desired Duration    Required Bandwidth
─────────────────────────────────────────────────────
10 GB          1 hour              ~22 Mbps
100 GB         4 hours             ~55 Mbps
1 TB           24 hours            ~93 Mbps
10 TB          7 days              ~132 Mbps
```

**Optimization:**

- Compression reduces transfer by 60-80%
- Colocate Sensei near source (or target if write-bound)
- Use direct peering for cloud-to-cloud

---

### Storage Requirements

| Storage Type        | Usage               | Sizing                             |
| ------------------- | ------------------- | ---------------------------------- |
| **Checkpoints**     | Resume capability   | ~0.1% of data volume               |
| **Logs**            | Operational records | ~1 GB/day/migration                |
| **Temp Files**      | Staging, sorting    | ~10% of largest table              |
| **Pattern Library** | Learning            | ~1 GB base + 10 MB/1000 migrations |

**Example sizing:**

```yaml
# 100 GB migration
storage:
  checkpoints: 100 MB × 5 retained = 500 MB
  logs: 1 GB/day × 3 days = 3 GB
  temp: 10 GB (largest table staging)
  total_temporary: ~14 GB
```

---

### GPU Requirements (Optional)

GPUs accelerate local LLM inference:

| Model Size  | GPU Memory     | Inference Speed |
| ----------- | -------------- | --------------- |
| Llama-3-8B  | 16 GB          | 50 tokens/sec   |
| Llama-3-70B | 80 GB (2×A100) | 20 tokens/sec   |
| Mistral-7B  | 14 GB          | 60 tokens/sec   |

**When to use GPUs:**

- High-volume semantic inference
- Air-gapped deployments (no API access)
- Cost optimization for heavy LLM usage

**GPU configuration:**

```yaml
resources:
  gpu:
    type: nvidia-a100-40gb
    count: 1
    model: llama-3-8b-instruct
    quantization: int8 # Reduce memory by 50%
```

---

### Resource Limits

Configure resource limits to prevent runaway usage:

```yaml
resource_limits:
  # Per-migration limits
  per_migration:
    max_workers: 32
    max_memory_gb: 64
    max_cpu_cores: 32
    max_temp_storage_gb: 100

  # Global limits
  global:
    max_concurrent_migrations: 20
    max_total_workers: 256
    max_total_memory_gb: 512
```

---

### Auto-Scaling

Sensei scales resources dynamically:

```yaml
autoscaling:
  enabled: true

  worker_scaling:
    min_workers: 4
    max_workers: 64
    target_cpu_utilization: 70%
    scale_up_cooldown: 60s
    scale_down_cooldown: 300s

  memory_scaling:
    initial_memory_per_worker: 1Gi
    max_memory_per_worker: 4Gi
    scale_on_oom: true
```

---

### Cost Estimation

Estimate compute costs:

```bash
curl https://api.sensei.ai/v1/migrations/{id}/cost-estimate
```

Response:

```json
{
  "estimate": {
    "compute_hours": 48,
    "memory_gb_hours": 768,
    "storage_gb_hours": 2400,
    "network_transfer_gb": 127,
    "llm_inference_tokens": 15000000,

    "cost_breakdown": {
      "compute": "$96.00",
      "memory": "$38.40",
      "storage": "$4.80",
      "network": "$11.43",
      "llm_inference": "$22.50",
      "platform_fee": "$50.00",
      "total": "$223.13"
    }
  }
}
```

→ [Performance Tuning](performance-tuning.md)
→ [Scheduling](scheduling.md)
→ [Operations Overview](README.md)
