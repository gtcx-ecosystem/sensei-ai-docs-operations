# Performance Tuning

## Optimization Techniques

Sensei auto-optimizes most performance parameters, but understanding the levers helps you tune for specific scenarios.

---

### Performance Factors

```
                         OVERALL THROUGHPUT
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
   SOURCE READ            TRANSFORMATION           TARGET WRITE
        │                       │                       │
   ┌────┴────┐             ┌────┴────┐             ┌────┴────┐
   │Network  │             │CPU      │             │Network  │
   │Query    │             │Memory   │             │Commit   │
   │Batch    │             │Workers  │             │Index    │
   └─────────┘             └─────────┘             └─────────┘
```

Throughput is limited by the slowest stage.

---

### Bottleneck Detection

Sensei automatically detects bottlenecks:

```yaml
bottleneck_analysis:
  current_bottleneck: target_write
  
  stages:
    source_read:
      throughput: 2.1M records/hr
      utilization: 45%
      status: healthy
      
    transformation:
      throughput: 3.5M records/hr
      utilization: 35%
      status: healthy
      
    target_write:
      throughput: 800K records/hr
      utilization: 95%
      status: bottleneck  # ← Limiting factor
      
  recommendation:
    action: increase_write_parallelism
    expected_improvement: 40%
    steps:
      - "Increase write batch size from 5000 to 10000"
      - "Enable parallel bulk loading"
      - "Defer index creation until after load"
```

View current bottleneck:
```bash
curl https://api.sensei.ai/v1/migrations/{id}/performance/bottleneck
```

---

### Source Read Tuning

| Parameter | Default | Effect | When to Adjust |
|-----------|---------|--------|----------------|
| `read_batch_size` | 10,000 | Rows per query | Increase for fast networks |
| `parallel_readers` | 4 | Concurrent read threads | Increase if source can handle |
| `query_timeout` | 3600s | Max query duration | Increase for slow sources |
| `fetch_size` | 10,000 | JDBC cursor size | Match batch size |

```yaml
source:
  performance:
    read_batch_size: 50000      # Large batches
    parallel_readers: 8          # More parallelism
    prefetch_next_batch: true   # Pipeline reads
```

**Source-specific tips:**
- **Oracle:** Use `/*+ PARALLEL(8) */` hints
- **PostgreSQL:** Use `cursor_tuple_fraction = 0.1`
- **MySQL:** Use `SET SESSION read_buffer_size = 16M`

---

### Transformation Tuning

| Parameter | Default | Effect | When to Adjust |
|-----------|---------|--------|----------------|
| `transform_workers` | 8 | Parallel transformation threads | CPU-bound workloads |
| `transform_batch_size` | 10,000 | Records per transform batch | Memory-constrained |
| `llm_cache_size` | 1000 | Cached LLM inference results | Repetitive patterns |

```yaml
transformation:
  performance:
    workers: 16                 # More parallelism
    vectorize: true             # Use SIMD operations
    cache_transforms: true      # Cache compiled transforms
    jit_compile: true           # JIT compile hot paths
```

**Tips:**
- Complex transformations benefit from more workers
- Simple type conversions are often memory-bound
- LLM caching dramatically helps repetitive patterns

---

### Target Write Tuning

| Parameter | Default | Effect | When to Adjust |
|-----------|---------|--------|----------------|
| `write_batch_size` | 5,000 | Rows per commit | Balance commit overhead vs. memory |
| `parallel_writers` | 4 | Concurrent write connections | Target can handle parallel |
| `defer_indexes` | true | Create indexes after load | Large tables |
| `bulk_load` | true | Use COPY/bulk insert | When available |

```yaml
target:
  performance:
    write_batch_size: 10000
    parallel_writers: 8
    defer_indexes: true
    defer_constraints: true
    disable_triggers: true
    use_bulk_load: true
```

**Target-specific tips:**
- **PostgreSQL:** Use `COPY`, set `synchronous_commit = off`
- **Snowflake:** Stage to S3, use `COPY INTO`
- **BigQuery:** Use load jobs, not streaming inserts
- **MySQL:** Use `LOAD DATA`, disable keys during load

---

### Strategy Presets

Quick presets for common scenarios:

#### `speed` — Maximize throughput
```yaml
strategy: speed
# Results in:
#   - Large batch sizes
#   - Maximum parallelism
#   - Deferred indexes/constraints
#   - Minimal checkpointing
#   - Lower error tolerance
```

#### `safe` — Minimize risk
```yaml
strategy: safe
# Results in:
#   - Small batch sizes
#   - Conservative parallelism
#   - Immediate index updates
#   - Frequent checkpointing
#   - Higher error tolerance
```

#### `balanced` — Default
```yaml
strategy: balanced
# Results in:
#   - Moderate batch sizes
#   - Adaptive parallelism
#   - Deferred indexes (configurable)
#   - Regular checkpointing
#   - Balanced error handling
```

---

### Adaptive Optimization

Sensei continuously adjusts during execution:

```yaml
adaptive_optimization:
  enabled: true
  
  behaviors:
    # Adjust batch sizes based on observed performance
    batch_size_tuning:
      enabled: true
      min_batch: 1000
      max_batch: 100000
      adjustment_interval: 5m
    
    # Spawn/reduce workers based on bottleneck
    worker_scaling:
      enabled: true
      min_workers: 2
      max_workers: 32
      scale_up_threshold: 0.8   # Utilization
      scale_down_threshold: 0.3
    
    # Adjust compression based on network
    compression_tuning:
      enabled: true
      min_compression: 0
      max_compression: 9
```

---

### Performance Benchmarks

Typical throughput by scenario:

| Scenario | Source | Target | Throughput |
|----------|--------|--------|------------|
| Simple type conversion | PostgreSQL | PostgreSQL | 2-3M rec/hr |
| Complex transformation | Oracle | Snowflake | 800K-1.2M rec/hr |
| Large blob data | SQL Server | S3+PostgreSQL | 200-400K rec/hr |
| High cardinality | MySQL | BigQuery | 1-2M rec/hr |

**Factors that slow throughput:**
- Complex transformations (regex, LLM inference)
- Large blob/text columns
- Many foreign key constraints
- Slow network between source and target

→ [Resource Management](resource-management.md)
→ [Monitoring](monitoring.md)
→ [Operations Overview](README.md)
