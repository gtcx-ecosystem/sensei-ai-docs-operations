# Scaling

## Horizontal and Vertical Scaling

Sensei scales to handle increasing migration workloads. This page covers scaling strategies, auto-scaling configuration, and capacity planning.

---

### Scaling Dimensions

| Dimension             | Scales With           | Approach                 |
| --------------------- | --------------------- | ------------------------ |
| Concurrent migrations | Business demand       | Horizontal (workers)     |
| Data volume           | Migration size        | Horizontal (workers)     |
| Throughput            | Speed requirements    | Horizontal + vertical    |
| Schema complexity     | LLM inference         | Vertical (LLM resources) |
| Pattern library       | Learning accumulation | Vertical (vector DB)     |

---

### Component Scaling

#### Workers (Primary Scaling Target)

Workers handle data movement — scale these first:

```yaml
# Manual scaling
kubectl scale deployment sensei-worker -n sensei --replicas=16

# Helm upgrade
helm upgrade sensei sensei/sensei \
  --set dataPlane.workers.replicas=16
```

**Scaling guidance:**

- 1-2 migrations: 4 workers
- 3-5 migrations: 8 workers
- 5-10 migrations: 16 workers
- 10-20 migrations: 32 workers
- 20+ migrations: 64+ workers

#### MABA (Transformation Engine)

Scale when transformation is the bottleneck:

```yaml
dataPlane:
  maba:
    replicas: 4 # Default: 2
    resources:
      requests:
        cpu: 16 # Default: 8
        memory: 32Gi # Default: 16Gi
```

#### KORA (Validation Engine)

Scale during validation phases:

```yaml
dataPlane:
  kora:
    replicas: 4
    resources:
      requests:
        cpu: 8
        memory: 16Gi
```

---

### Auto-Scaling

#### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sensei-worker-hpa
  namespace: sensei
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sensei-worker
  minReplicas: 4
  maxReplicas: 64
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 120
```

#### Custom Metrics Scaling

Scale based on migration-specific metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sensei-worker-custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sensei-worker
  minReplicas: 4
  maxReplicas: 64
  metrics:
    - type: External
      external:
        metric:
          name: sensei_pending_batches
        target:
          type: AverageValue
          averageValue: 10
```

#### Helm Configuration

```yaml
dataPlane:
  workers:
    autoscaling:
      enabled: true
      minReplicas: 4
      maxReplicas: 64
      targetCPUUtilization: 70
      targetMemoryUtilization: 80
      # Custom metrics
      customMetrics:
        - name: pending_batches
          target: 10
```

---

### Cluster Auto-Scaling

Scale the underlying cluster nodes:

#### AWS EKS

```yaml
# Cluster Autoscaler
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          command:
            - ./cluster-autoscaler
            - --cloud-provider=aws
            - --nodes=3:20:sensei-workers
            - --scale-down-delay-after-add=5m
            - --scale-down-unneeded-time=5m
```

#### GKE

```bash
# Enable cluster autoscaling
gcloud container clusters update sensei-cluster \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=20 \
  --node-pool=worker-pool
```

---

### Vertical Scaling

For components that benefit from larger instances:

#### Database (PostgreSQL)

```yaml
postgresql:
  primary:
    resources:
      requests:
        cpu: 8 # Scale up for heavy workloads
        memory: 32Gi
```

#### Vector Database

```yaml
vectordb:
  resources:
    requests:
      cpu: 4
      memory: 16Gi
  persistence:
    size: 200Gi # Scale as pattern library grows
```

#### LLM Inference

```yaml
llm:
  local:
    gpu:
      count: 2 # Add GPUs for faster inference
    resources:
      memory: 80Gi
```

---

### Scaling Triggers

| Metric                  | Threshold  | Action             |
| ----------------------- | ---------- | ------------------ |
| Worker CPU > 70%        | 5 minutes  | Scale up workers   |
| Worker CPU < 30%        | 10 minutes | Scale down workers |
| Pending batches > 100   | 2 minutes  | Scale up workers   |
| MABA queue depth > 50   | 5 minutes  | Scale up MABA      |
| Validation backlog > 20 | 5 minutes  | Scale up KORA      |

---

### Scaling Best Practices

1. **Start with horizontal** — Add more workers before making instances bigger
2. **Use auto-scaling** — Let the system respond to demand automatically
3. **Set appropriate limits** — Don't scale beyond infrastructure capacity
4. **Monitor scaling events** — Watch for thrashing (rapid scale up/down)
5. **Plan for bursts** — Keep some headroom for sudden demand
6. **Test at scale** — Verify performance at expected maximum load

---

### Cost Optimization

Balance performance and cost:

```yaml
# Spot instances for workers (AWS)
nodeSelector:
  node.kubernetes.io/instance-type: spot

# Preemptible VMs (GCP)
nodeSelector:
  cloud.google.com/gke-preemptible: "true"

# Workers can handle interruption via checkpointing
dataPlane:
  workers:
    tolerations:
    - key: "spot"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

→ [Infrastructure Requirements](infrastructure.md)
→ [Performance Tuning](../migrations/performance-tuning.md)
→ [Deployment Overview](README.md)
