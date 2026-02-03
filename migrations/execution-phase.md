# Execution Phase

## Running the Migration

The execution phase is where data actually moves from source to target. Worker Agents read, transform, and write data while Validator Agents verify each batch in real-time.

---

### Execution Flow

```text
┌────────────────────────────────────────────────────────────────────┐
│                       EXECUTION PHASE                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  For each phase in DAG:                                            │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  For each table (parallel if allowed):                       │  │
│  │  ┌───────────────────────────────────────────────────────┐  │  │
│  │  │  While rows remaining:                                 │  │  │
│  │  │                                                        │  │  │
│  │  │  1. Read batch (10K rows)                             │  │  │
│  │  │  2. Transform batch (apply mappings)                  │  │  │
│  │  │  3. Validate batch (inline checks)                    │  │  │
│  │  │  4. Handle errors (recover or queue)                  │  │  │
│  │  │  5. Write batch (bulk load)                           │  │  │
│  │  │  6. Checkpoint (every N batches)                      │  │  │
│  │  │  7. Report progress                                   │  │  │
│  │  │                                                        │  │  │
│  │  └───────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

### Starting Execution

**Via API:**

```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/start \
  -H "Authorization: Bearer sk_live_abc123"
```

**Via SDK:**

```python
migration = client.migrations.get("mig_abc123")
migration.start()
```

**Via Dashboard:**
Click "Start Migration" on the approved migration plan.

---

### Execution Monitoring

#### Real-Time Metrics

| Metric              | Description                 | Typical Range    |
| ------------------- | --------------------------- | ---------------- |
| `records_processed` | Total rows migrated         | 0 → total        |
| `records_per_hour`  | Current throughput          | 500K - 2M        |
| `errors_total`      | All errors encountered      | Varies           |
| `errors_recovered`  | Auto-recovered errors       | 90-99% of errors |
| `errors_escalated`  | Errors needing human review | 0-10% of errors  |
| `current_phase`     | Active DAG phase            | 1 → N            |
| `current_table`     | Table(s) being processed    | Name(s)          |
| `progress_percent`  | Overall completion          | 0-100%           |
| `eta_completion`    | Estimated finish time       | DateTime         |

#### Progress API

```bash
curl https://api.sensei.ai/v1/migrations/{id}/status
```

Response:

```json
{
  "status": "running",
  "phase": {
    "current": 3,
    "total": 5,
    "name": "Transaction Data"
  },
  "progress": {
    "percent": 62,
    "records_processed": 32456789,
    "records_total": 52000000,
    "throughput_per_hour": 1250000
  },
  "timing": {
    "started_at": "2026-02-02T08:00:00Z",
    "elapsed_hours": 12.5,
    "eta_completion": "2026-02-02T20:30:00Z"
  },
  "errors": {
    "total": 847,
    "recovered": 834,
    "escalated": 13,
    "recovery_rate": 0.985
  }
}
```

---

### Execution Configuration

```yaml
execution:
  # Parallelism
  parallel:
    max_workers: 16 # Worker Agent instances
    tables_parallel: true # Process independent tables together
    max_tables_concurrent: 4 # Limit concurrent table processing

  # Batching
  batching:
    read_batch_size: 10000
    write_batch_size: 5000
    transform_batch_size: 10000

  # Checkpointing
  checkpointing:
    interval_records: 100000 # Checkpoint every 100K records
    interval_minutes: 15 # Or every 15 minutes
    keep_last_n: 5 # Retain last 5 checkpoints

  # Error handling
  errors:
    auto_recover: true
    max_consecutive_errors: 100
    escalation_threshold: 0.01 # Escalate if >1% error rate
    pause_on_escalation: false

  # Performance
  performance:
    adaptive_batch_sizing: true
    bottleneck_detection: true
    auto_scale_workers: true
```

---

### Pause and Resume

**Pause:**

```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/pause
```

Pausing:

- Completes the current batch
- Writes a checkpoint
- Stops reading new batches
- Releases worker resources

**Resume:**

```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/resume
```

Resuming:

- Loads the last checkpoint
- Reacquires worker resources
- Continues from where it stopped

---

### Common Scenarios

#### Scenario 1: Smooth Execution

```text
Phase 1 (Reference): ████████████ 100% (5 min)
Phase 2 (Customers): ████████████ 100% (1.5 hr)
Phase 3 (Orders):    ████████████ 100% (16 hr)
Phase 4 (Items):     ████████████ 100% (4 hr)
Phase 5 (Archive):   ████████████ 100% (30 min)

Total: 22 hours, 847 errors auto-recovered, 0 escalated
```

#### Scenario 2: Error Spike (Auto-Handled)

```text
Phase 3 (Orders):    ██████░░░░░░ 52%
  └─ Error spike detected: 150 encoding errors in batch
  └─ Auto-recovery: Applied UTF-8 normalization
  └─ Swarm update: All workers now use fix proactively
  └─ Continuing...
Phase 3 (Orders):    ████████████ 100%
```

#### Scenario 3: Escalation Required

```text
Phase 3 (Orders):    ████████░░░░ 67%
  └─ Error escalated: 234 records fail FK constraint
  └─ Cause: Customer records missing from phase 2
  └─ Action required: Review error_queue_orders_fk
  └─ Migration PAUSED pending resolution
```

---

### Performance Optimization

Sensei automatically optimizes execution, but you can tune:

| Lever                 | Effect               | When to Adjust               |
| --------------------- | -------------------- | ---------------------------- |
| `max_workers`         | More parallelism     | Large datasets, fast network |
| `read_batch_size`     | Fewer DB round-trips | Low-latency source           |
| `write_batch_size`    | Fewer commits        | High-throughput target       |
| `checkpoint_interval` | Less I/O overhead    | Stable environment           |

→ [Performance Tuning](performance-tuning.md)
→ [Error Handling](error-handling.md)
→ [Checkpointing](checkpointing.md)
