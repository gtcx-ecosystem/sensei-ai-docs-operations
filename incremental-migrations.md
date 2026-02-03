# Incremental Migrations

## Ongoing Sync and Change Data Capture

Incremental migrations keep source and target synchronized over time — essential for zero-downtime cutovers and ongoing data replication.

---

### When to Use Incremental

| Scenario | Migration Type |
|----------|---------------|
| Source is static during migration | Full migration |
| Source changes during migration | Incremental required |
| Zero-downtime cutover needed | Incremental required |
| Ongoing replication (not migration) | Incremental (continuous) |
| Large dataset, limited window | Incremental (multi-pass) |

---

### How Incremental Works

```
INITIAL LOAD
─────────────────────────────────────────────
Source (T0)                    Target
┌────────────┐                ┌────────────┐
│ A B C D E  │ ─── Full ────→ │ A B C D E  │
└────────────┘                └────────────┘

INCREMENTAL SYNC (T1)
─────────────────────────────────────────────
Source (T1)                    Target
┌────────────┐                ┌────────────┐
│ A B' C D E │                │ A B C D E  │
│     F      │                │            │
│  (G deleted)                │            │
└────────────┘                └────────────┘
       │                             
       │    CDC captures: UPDATE B, INSERT F, DELETE G
       ↓                             
┌────────────┐
│ A B' C D E │ ─── Changes ──→ Target updated
│ F          │
└────────────┘
```

---

### CDC Sources

Sensei captures changes via:

| Database | CDC Method | Latency |
|----------|-----------|---------|
| PostgreSQL | Logical replication | Real-time |
| MySQL/MariaDB | Binary log | Real-time |
| Oracle | LogMiner / GoldenGate | Real-time |
| SQL Server | Change Tracking / CDC | Real-time |
| MongoDB | Change Streams | Real-time |
| Generic | Timestamp/version column | Batch (minutes) |

---

### Configuration

```yaml
migration:
  type: incremental
  
  initial_load:
    enabled: true
    # or: skip (if target already has baseline)
  
  cdc:
    method: auto  # Auto-detect best method
    # or: logical_replication, binlog, timestamp_column
    
    # For timestamp-based CDC
    timestamp_column: updated_at
    
    # Capture frequency
    poll_interval: 30s  # For timestamp-based
    
    # Conflict resolution
    conflict_resolution: source_wins  # or target_wins, manual
  
  continuous:
    enabled: true
    # Run until manually stopped
    # or: run until source/target in sync, then stop
    stop_when: manual  # or in_sync
```

---

### Three-Phase Pattern

For zero-downtime migrations:

#### Phase 1: Initial Load
Migrate all existing data while source continues operating.

```python
migration = client.migrations.create(
    type="incremental",
    initial_load=True
)
migration.start()

# Source continues normal operation
# Target catches up with historical data
```

#### Phase 2: Catch-Up
Apply accumulated changes until target is nearly caught up.

```python
# Continuous sync runs automatically
# Monitor lag until acceptable

while migration.lag_seconds > 60:
    time.sleep(10)

# Lag is now under 1 minute
```

#### Phase 3: Cutover
Brief source freeze, final sync, then switch.

```python
# Freeze source writes
source.freeze_writes()

# Wait for final sync
migration.wait_for_sync()

# Verify sync complete
assert migration.lag_seconds == 0

# Cutover to target
switch_traffic_to_target()

# Unfreeze source (now backup)
source.unfreeze_writes()
```

---

### Lag Monitoring

Track how far behind target is:

```bash
curl https://api.sensei.ai/v1/migrations/{id}/lag
```

```json
{
  "lag": {
    "seconds": 45,
    "records": 12847,
    "bytes": 15728640
  },
  "trend": "decreasing",
  "estimated_catch_up": "2 minutes",
  "cdc_position": {
    "postgresql": {
      "lsn": "0/16B3748",
      "timestamp": "2026-02-02T14:23:45Z"
    }
  }
}
```

**Lag alerts:**
```yaml
alerting:
  rules:
    - name: high_cdc_lag
      condition: lag_seconds > 300
      severity: high
      message: "CDC lag exceeded 5 minutes"
```

---

### Conflict Resolution

When the same record is modified in both source and target:

| Strategy | Behavior |
|----------|----------|
| `source_wins` | Source change overwrites target |
| `target_wins` | Target change preserved |
| `newest_wins` | Compare timestamps, newest wins |
| `manual` | Queue for human review |
| `custom` | Apply custom merge logic |

```yaml
conflict_resolution:
  default: source_wins
  
  # Per-table overrides
  overrides:
    - table: user_preferences
      strategy: target_wins  # User changes on target preserved
      
    - table: inventory
      strategy: newest_wins
      timestamp_column: last_updated
```

---

### Stopping Incremental

```bash
# Stop gracefully (finish current batch)
curl -X POST https://api.sensei.ai/v1/migrations/{id}/stop

# Stop immediately (may lose in-flight changes)
curl -X POST https://api.sensei.ai/v1/migrations/{id}/stop?immediate=true
```

**When to stop:**
- Cutover complete, source decommissioned
- Migration converted to full sync (different tool)
- Testing/validation complete

---

### Performance

| Factor | Impact on CDC |
|--------|--------------|
| Change volume | More changes = more processing |
| Network latency | Higher latency = higher lag |
| Target write speed | Slow target = lag accumulation |
| Transformation complexity | Complex transforms = slower application |

**Typical throughput:**
- Simple CDC: 10,000-50,000 changes/second
- Complex transforms: 1,000-5,000 changes/second

---

### Best Practices

1. **Test cutover procedure** — Practice the freeze-sync-switch sequence
2. **Monitor lag continuously** — Set alerts for unacceptable lag
3. **Plan for catch-up time** — Initial load creates lag; plan time to catch up
4. **Test conflict resolution** — Verify conflicts are handled as expected
5. **Have rollback plan** — Know how to switch back to source

→ [Cutover Strategies](cutover-strategies.md)
→ [Scheduling](scheduling.md)
→ [Operations Overview](README.md)
