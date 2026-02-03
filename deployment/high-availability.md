# High Availability

## Redundancy, Failover, and Disaster Recovery

Sensei is designed for high availability. This page covers HA architecture, failure scenarios, and disaster recovery planning.

---

### HA Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    AVAILABILITY ZONE A                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │ API (1)  │  │ MABA (1) │  │ Worker   │  │ PostgreSQL│            │
│  │          │  │          │  │ (1,2,3,4)│  │ (Primary) │            │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │
└─────────────────────────────────────────────────────────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │   Load Balancer     │
                    └──────────┬──────────┘
                               │
┌─────────────────────────────────────────────────────────────────────┐
│                    AVAILABILITY ZONE B                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │ API (2)  │  │ MABA (2) │  │ Worker   │  │ PostgreSQL│            │
│  │          │  │          │  │ (5,6,7,8)│  │ (Standby) │            │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Component Redundancy

| Component    | Minimum for HA      | Failure Impact      | Recovery Time |
| ------------ | ------------------- | ------------------- | ------------- |
| API Server   | 2                   | Degraded API access | Instant (LB)  |
| Dashboard    | 2                   | Degraded UI access  | Instant (LB)  |
| Orchestrator | 2                   | Workflow delays     | <30s failover |
| MABA         | 2                   | Reduced throughput  | Instant       |
| KORA         | 2                   | Validation delays   | Instant       |
| Workers      | 4+                  | Reduced throughput  | Instant       |
| PostgreSQL   | 2 (primary+standby) | Write unavailable   | <60s failover |
| Redis        | 3 (sentinel)        | Cache miss          | <30s failover |

---

### Database HA

#### PostgreSQL

```yaml
postgresql:
  replication:
    enabled: true
    numSynchronousReplicas: 1

  primary:
    resources:
      requests:
        cpu: 4
        memory: 8Gi

  readReplicas:
    replicaCount: 2
```

**Failover behavior:**

- Automatic failover via Patroni (bundled)
- Writes unavailable during failover (~30-60 seconds)
- Reads continue from replicas

#### Redis

```yaml
redis:
  sentinel:
    enabled: true
    quorum: 2

  replica:
    replicaCount: 2
```

**Failover behavior:**

- Sentinel detects primary failure
- Promotes replica to primary
- Clients reconnect automatically

---

### Stateless Components

API, Dashboard, MABA, KORA, Workers, and Agents are stateless:

- Run multiple replicas
- Load balancer distributes traffic
- Any instance can handle any request
- Failed instance replaced automatically

```yaml
# Example: API deployment with HA
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sensei-api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: sensei-api
                topologyKey: topology.kubernetes.io/zone
```

---

### Failure Scenarios

#### Scenario 1: Single Pod Failure

**Impact:** None  
**Recovery:** Kubernetes restarts pod, LB routes around

#### Scenario 2: Single Node Failure

**Impact:** Temporary capacity reduction  
**Recovery:** Pods rescheduled to other nodes (1-2 minutes)

#### Scenario 3: Availability Zone Failure

**Impact:** 50% capacity reduction  
**Recovery:** Traffic routed to surviving AZ (instant)

#### Scenario 4: Primary Database Failure

**Impact:** Writes unavailable  
**Recovery:** Automatic failover (30-60 seconds)

#### Scenario 5: Redis Primary Failure

**Impact:** Cache misses, temporary latency increase  
**Recovery:** Sentinel promotes replica (10-30 seconds)

#### Scenario 6: Complete Region Failure

**Impact:** Service unavailable  
**Recovery:** DR activation required (see below)

---

### Disaster Recovery

#### RPO and RTO Targets

| Tier     | RPO        | RTO        | Configuration                    |
| -------- | ---------- | ---------- | -------------------------------- |
| Standard | 1 hour     | 4 hours    | Daily backups, single region     |
| Enhanced | 15 minutes | 1 hour     | Frequent backups, standby region |
| Premium  | Near-zero  | 15 minutes | Multi-region active-active       |

#### DR Configuration

**Standby Region (Enhanced):**

```yaml
disasterRecovery:
  enabled: true
  mode: standby

  primaryRegion: us-east-1
  drRegion: us-west-2

  replication:
    postgresql:
      enabled: true
      mode: async
      lagThreshold: 60s

    objectStorage:
      enabled: true
      mode: cross-region-replication
```

**Active-Active (Premium):**

```yaml
disasterRecovery:
  enabled: true
  mode: active-active

  regions:
    - us-east-1
    - us-west-2

  routing:
    type: latency-based
    healthCheck:
      interval: 10s
      threshold: 3
```

---

### Failover Procedures

#### Automated Failover (Component Level)

Handled automatically by:

- Kubernetes (pod/node failures)
- Load balancer (instance failures)
- Database HA (primary failures)

#### Manual Failover (Region Level)

```bash
# 1. Verify DR region is healthy
sensei dr status --region us-west-2

# 2. Initiate failover
sensei dr failover \
  --from us-east-1 \
  --to us-west-2 \
  --confirm

# 3. Update DNS (if not using global LB)
# 4. Verify services in DR region
# 5. Notify stakeholders
```

---

### Testing

Regular HA testing is essential:

| Test         | Frequency | Procedure                        |
| ------------ | --------- | -------------------------------- |
| Pod failure  | Monthly   | Kill random pod, verify recovery |
| Node failure | Quarterly | Drain node, verify rescheduling  |
| AZ failure   | Quarterly | Simulate AZ outage in test env   |
| DR failover  | Annually  | Full DR drill                    |

```bash
# Chaos testing with Chaos Mesh
kubectl apply -f chaos-pod-kill.yaml

# Verify system remains operational
sensei health check
```

→ [Backup & Recovery](backup-recovery.md)
→ [Scaling](scaling.md)
→ [Deployment Overview](README.md)
