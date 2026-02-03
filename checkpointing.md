# Checkpointing

## Resume From Any Failure Point

Checkpointing enables migrations to resume from where they stopped — whether due to failure, manual pause, or infrastructure issues. Sensei never loses more than one batch of work.

---

### How Checkpointing Works

```
Migration Execution
      │
      ├── Batch 1 ─────→ Success ─→ [Checkpoint 1]
      │
      ├── Batch 2 ─────→ Success
      │
      ├── Batch 3 ─────→ Success ─→ [Checkpoint 2]
      │
      ├── Batch 4 ─────→ Success
      │
      ├── Batch 5 ─────→ FAILURE ────────────────┐
      │                                           │
      │                                           │
      └── Resume from [Checkpoint 2] ←────────────┘
          └── Batch 4 (re-process)
          └── Batch 5 (re-process)
          └── Continue...
```

---

### Checkpoint Contents

Each checkpoint captures:

```yaml
checkpoint:
  id: ckpt_abc123
  migration_id: mig_xyz789
  timestamp: "2026-02-02T14:23:00Z"
  
  # Position in migration
  position:
    phase: 3
    table: orders
    batch_number: 1547
    last_processed_id: 47291000
    records_completed: 15470000
  
  # State of all tables
  table_states:
    - table: customers
      status: completed
      records: 1250000
      
    - table: orders
      status: in_progress
      records_completed: 15470000
      records_total: 45000000
      last_id: 47291000
      
    - table: order_items
      status: pending
      
  # Error state
  error_state:
    error_count: 234
    error_queue: [list of pending errors]
    recovery_patterns_applied: [list]
    
  # Configuration snapshot
  config_snapshot:
    batch_size: 10000
    workers: 16
    strategy: balanced
```

---

### Checkpoint Configuration

```yaml
checkpointing:
  # When to checkpoint
  triggers:
    # Checkpoint every N records
    record_interval: 100000
    
    # Checkpoint every N minutes
    time_interval: 15
    
    # Checkpoint at phase boundaries
    phase_boundaries: true
    
    # Checkpoint before risky operations
    before_risky: true
  
  # Retention
  retention:
    keep_last_n: 5
    keep_all_phase_boundaries: true
    max_age_hours: 168  # 7 days
  
  # Storage
  storage:
    type: s3
    bucket: sensei-checkpoints
    encryption: AES-256
    compress: true
```

---

### Checkpoint Operations

#### List Checkpoints
```bash
curl https://api.sensei.ai/v1/migrations/{id}/checkpoints
```

Response:
```json
{
  "checkpoints": [
    {
      "id": "ckpt_001",
      "timestamp": "2026-02-02T14:00:00Z",
      "type": "automatic",
      "phase": 3,
      "records_completed": 15000000,
      "size_bytes": 245760
    },
    {
      "id": "ckpt_002",
      "timestamp": "2026-02-02T14:15:00Z",
      "type": "automatic",
      "phase": 3,
      "records_completed": 15500000,
      "size_bytes": 251904
    }
  ]
}
```

#### Create Manual Checkpoint
```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/checkpoints \
  -d '{"reason": "Before risky manual operation"}'
```

#### Restore from Checkpoint
```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/restore \
  -d '{"checkpoint_id": "ckpt_001"}'
```

---

### Resume Scenarios

#### Scenario 1: Process Crash
```
Migration running ─→ Worker process crashes
                           │
                           ↓
        System detects orphaned migration
                           │
                           ↓
        Loads latest checkpoint (ckpt_002)
                           │
                           ↓
        Resumes from batch 1551 (lost 3 batches)
```

#### Scenario 2: Manual Pause
```
Migration running ─→ User pauses migration
                           │
                           ↓
        Complete current batch
                           │
                           ↓
        Create checkpoint (current state)
                           │
                           ↓
        Release resources, enter paused state
                           │
        ... time passes ...
                           │
        User resumes ─→ Load checkpoint ─→ Continue
```

#### Scenario 3: Infrastructure Failure
```
Migration running ─→ Database connection lost
                           │
                           ↓
        Current batch fails
                           │
                           ↓
        Retry connection (exponential backoff)
                           │
        ┌──────────────────┴──────────────────┐
        │                                      │
    Success                               Exhausted
        │                                      │
        ↓                                      ↓
    Continue                           Pause migration
    (no checkpoint                     Create checkpoint
     restore needed)                   Alert operators
```

---

### Data Loss Guarantee

**Maximum data loss:** One batch (default 10,000 records)

Worst case: Failure occurs immediately after a batch completes but before checkpoint. The batch must be re-processed, but it's idempotent (same result on re-run).

**For zero data loss:** Set `checkpoint_after_every_batch: true` (impacts performance).

---

### Checkpoint Storage

| Plan | Storage Location | Retention | Encryption |
|------|-----------------|-----------|------------|
| Starter | Sensei-managed S3 | 7 days | AES-256 |
| Professional | Sensei-managed S3 | 30 days | AES-256 |
| Enterprise | Your S3/GCS/Azure | Configurable | Your KMS |

**Checkpoint sizes:**
- Small migration: 50-200 KB per checkpoint
- Medium migration: 200 KB - 2 MB per checkpoint
- Large migration: 2-20 MB per checkpoint

---

### Troubleshooting

#### Checkpoint Restoration Fails
```yaml
error: "Checkpoint ckpt_001 refers to source row_id 47291000, but source now starts at 50000000"
cause: "Source data was modified after checkpoint was created"
resolution:
  - If source was truncated: Restart migration from beginning
  - If source had new rows: Continue from checkpoint (new rows will be picked up)
  - If source had deletions: May cause FK issues; review and decide
```

#### Checkpoint Too Old
```yaml
error: "Checkpoint ckpt_001 is 72 hours old; significant source drift possible"
warning: true
options:
  - Continue from checkpoint (accept potential drift)
  - Restart migration with fresh analysis
  - Run validation to check drift magnitude
```

→ [Rollback](rollback.md)
→ [Error Handling](error-handling.md)
→ [Operations Overview](README.md)
