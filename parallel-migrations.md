# Parallel Migrations

## Running Multiple Migrations Simultaneously

Organizations often need to run multiple migrations concurrently — different databases, different target environments, or phased rollouts. Sensei supports parallel migrations with resource isolation and intelligent scheduling.

---

### Concurrency Limits

| Plan | Concurrent Migrations | Shared Resources |
|------|----------------------|------------------|
| Starter | 2 | Agent pool, LLM inference |
| Professional | 10 | Agent pool, LLM inference |
| Enterprise | Unlimited | Dedicated pools available |

---

### Resource Allocation

Each migration gets isolated resources by default:

```yaml
# Per-migration resource allocation
migration_resources:
  migration_1:
    workers: 8
    memory_gb: 16
    
  migration_2:
    workers: 8
    memory_gb: 16
    
  # Shared resources
  shared:
    llm_inference: true
    pattern_library: true
    agent_coordinator: true
```

**Resource contention:**
- LLM inference is shared but queued (no interference)
- Pattern library reads are parallel (no contention)
- Network may contend if migrations share source/target

---

### Isolation Levels

#### Default: Shared Cluster
Migrations share the Sensei cluster but have isolated workers:

```yaml
isolation: shared
# Pros: Cost-efficient, pattern library sharing
# Cons: Resource contention possible
```

#### Dedicated Pools (Enterprise)
Each migration gets dedicated worker pools:

```yaml
isolation: dedicated
resources:
  worker_pool: dedicated_pool_1
  memory_quota: 64Gi
  cpu_quota: 32
# Pros: No contention, predictable performance
# Cons: Higher cost, no resource sharing
```

#### Fully Isolated (Enterprise)
Separate Sensei instances:

```yaml
isolation: instance
instance_id: sensei-team-a
# Pros: Complete isolation, compliance requirements
# Cons: Highest cost, no pattern library sharing
```

---

### Coordinating Multiple Migrations

#### Independent Migrations
When migrations don't interact:

```python
# Start multiple migrations in parallel
migrations = [
    client.migrations.create(source=source_a, target=target_a),
    client.migrations.create(source=source_b, target=target_b),
    client.migrations.create(source=source_c, target=target_c),
]

for mig in migrations:
    mig.start()

# Wait for all to complete
results = [mig.wait() for mig in migrations]
```

#### Dependent Migrations
When migrations must run in sequence:

```python
# Reference data first
ref_migration = client.migrations.create(
    name="Reference Data",
    source=ref_source,
    target=ref_target
)
ref_migration.start()
ref_migration.wait()

# Then dependent migrations in parallel
dependent_migrations = [
    client.migrations.create(
        name="Customers",
        dependencies=[ref_migration.id]  # Wait for ref data
    ),
    client.migrations.create(
        name="Products",
        dependencies=[ref_migration.id]
    ),
]
```

#### Phased Rollout
Migration per region:

```python
regions = ["us-east", "us-west", "eu", "apac"]

for region in regions:
    mig = client.migrations.create(
        name=f"Migration - {region}",
        source=sources[region],
        target=targets[region]
    )
    mig.start()
    result = mig.wait()
    
    if result.status != "certified":
        # Stop rollout on failure
        break
    
    # Proceed to next region
```

---

### Monitoring Multiple Migrations

#### Dashboard View
The dashboard shows all active migrations:

```
┌─────────────────────────────────────────────────────────────────┐
│ Active Migrations                                        3 of 10│
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ ● Migration A (Oracle → PostgreSQL)                             │
│   ████████████░░░░░░░░ 62% · Phase 3/5 · 847K rec/hr           │
│                                                                  │
│ ● Migration B (MySQL → Snowflake)                               │
│   ██████░░░░░░░░░░░░░░ 31% · Phase 2/4 · 1.2M rec/hr           │
│                                                                  │
│ ● Migration C (SQL Server → BigQuery)                           │
│   ████████████████████ 100% · Validating · ETA 2hr             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### API: List Active Migrations
```bash
curl "https://api.sensei.ai/v1/migrations?status=running"
```

#### Aggregate Metrics
```bash
curl https://api.sensei.ai/v1/metrics/aggregate
```

```json
{
  "active_migrations": 3,
  "total_records_migrated_today": 127500000,
  "total_errors_today": 2341,
  "total_errors_recovered": 2298,
  "aggregate_throughput_per_hour": 3847000,
  "resource_utilization": {
    "workers": "47/128",
    "memory": "156/512 GB",
    "cpu": "38/64 cores"
  }
}
```

---

### Resource Contention

When migrations compete for resources:

| Resource | Contention Symptom | Mitigation |
|----------|-------------------|------------|
| **CPU** | Slow transformations | Limit workers per migration |
| **Memory** | OOM errors | Reduce batch sizes |
| **Network** | Slow transfers | Stagger heavy phases |
| **Source DB** | Query timeouts | Limit concurrent readers |
| **Target DB** | Write timeouts | Limit concurrent writers |

**Automatic contention management:**
```yaml
contention_management:
  enabled: true
  
  # Automatically reduce parallelism when contention detected
  auto_throttle: true
  
  # Priority ordering (higher = more resources)
  priorities:
    mig_critical: 100
    mig_normal: 50
    mig_background: 10
```

---

### Best Practices

1. **Stagger start times** — Don't start all migrations simultaneously
2. **Separate source loads** — Don't hit the same source from multiple migrations
3. **Monitor aggregate resources** — Watch cluster-wide utilization
4. **Use priorities** — Mark critical migrations as higher priority
5. **Plan for failures** — One migration failing shouldn't cascade

→ [Scheduling](scheduling.md)
→ [Resource Management](resource-management.md)
→ [Operations Overview](README.md)
