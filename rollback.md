# Rollback

## Undo Strategies

Rollback capability is essential for production migrations. Sensei provides multiple rollback strategies depending on your requirements and constraints.

---

### Rollback Strategies

| Strategy | Speed | Data Loss | Complexity | When to Use |
|----------|-------|-----------|------------|-------------|
| **Target Truncate** | Fast | Target only | Low | Early phases, empty target |
| **Reverse Migration** | Slow | None | High | Production with bidirectional sync |
| **Point-in-Time** | Fast | To checkpoint | Medium | When PITR is available |
| **Backup Restore** | Medium | To backup | Low | Pre-migration backup taken |

---

### Strategy 1: Target Truncate

Simply delete all migrated data from the target.

**Best for:**
- Testing and staging
- Early migration phases
- Target database was empty before migration

**How it works:**
```sql
-- For each migrated table, in reverse dependency order
TRUNCATE TABLE order_items CASCADE;
TRUNCATE TABLE orders CASCADE;
TRUNCATE TABLE customers CASCADE;
```

**API:**
```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/rollback \
  -d '{"strategy": "truncate"}'
```

**Caution:** Destroys all data in migrated tables. Only use when target data loss is acceptable.

---

### Strategy 2: Reverse Migration

Migrate data back from target to source, effectively undoing the migration.

**Best for:**
- Production cutover failures
- When source must be restored to exact state

**How it works:**
```
1. Sensei creates reverse migration plan (target → source)
2. Applies reverse transformations
3. Migrates data back to source
4. Verifies source matches pre-migration state
```

**API:**
```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/rollback \
  -d '{
    "strategy": "reverse_migration",
    "verify_after": true
  }'
```

**Requirements:**
- Source must be writable
- Transformations must be reversible
- Time proportional to migration duration

---

### Strategy 3: Point-in-Time Recovery

Use database-native PITR to restore target to pre-migration state.

**Best for:**
- Cloud databases with PITR enabled
- Fast rollback needs

**Supported targets:**
- PostgreSQL (with WAL archiving)
- Snowflake (Time Travel)
- BigQuery (Time Travel)
- Oracle (Flashback)
- SQL Server (with full recovery model)

**How it works:**
```sql
-- Snowflake example
ALTER TABLE customers
  AT (TIMESTAMP => '2026-02-02 08:00:00'::timestamp_tz);

-- PostgreSQL example  
-- Requires pg_restore from WAL backup
```

**API:**
```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/rollback \
  -d '{
    "strategy": "point_in_time",
    "restore_to": "2026-02-02T07:59:00Z"
  }'
```

---

### Strategy 4: Backup Restore

Restore target from pre-migration backup.

**Best for:**
- When explicit backup was taken
- When PITR isn't available

**How it works:**
1. Sensei can take pre-migration backup (if configured)
2. On rollback, restore from backup
3. Verify restoration

**Configuration:**
```yaml
rollback:
  pre_migration_backup:
    enabled: true
    type: logical  # or physical
    storage: s3://backups/pre-migration/
    retention_days: 30
```

**API:**
```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/rollback \
  -d '{
    "strategy": "backup_restore",
    "backup_id": "bkp_abc123"
  }'
```

---

### Rollback Planning

Configure rollback options during migration planning:

```yaml
migration:
  rollback:
    # Enable automatic pre-migration backup
    backup_before_start: true
    backup_type: logical
    backup_location: s3://backups/
    
    # Keep rollback capability window
    rollback_window_hours: 168  # 7 days
    
    # Preferred rollback strategy
    preferred_strategy: point_in_time
    fallback_strategy: reverse_migration
    
    # Validation after rollback
    verify_rollback: true
```

---

### Partial Rollback

Rollback specific phases or tables:

```bash
# Rollback only phase 3
curl -X POST https://api.sensei.ai/v1/migrations/{id}/rollback \
  -d '{
    "strategy": "truncate",
    "scope": "phase",
    "phase": 3
  }'

# Rollback specific tables
curl -X POST https://api.sensei.ai/v1/migrations/{id}/rollback \
  -d '{
    "strategy": "truncate",
    "scope": "tables",
    "tables": ["orders", "order_items"]
  }'
```

---

### Rollback Verification

After rollback, verify the result:

```bash
curl https://api.sensei.ai/v1/migrations/{id}/rollback/status
```

Response:
```json
{
  "rollback_status": "completed",
  "strategy_used": "point_in_time",
  "restored_to": "2026-02-02T07:59:00Z",
  "verification": {
    "source_comparison": "passed",
    "record_counts_match": true,
    "sample_verification": "passed"
  },
  "duration_seconds": 847
}
```

---

### Rollback Limitations

| Limitation | Description | Mitigation |
|-----------|-------------|------------|
| **Downstream dependencies** | Other systems may have consumed migrated data | Coordinate rollback across systems |
| **Sequence gaps** | Auto-increment sequences may have gaps | Reset sequences after rollback |
| **Audit trails** | Audit logs may reference rolled-back transactions | Document rollback in audit |
| **Non-reversible transforms** | Some transformations lose information | Plan for this during mapping |

---

### Rollback Runbook

Standard procedure for rollback:

1. **Assess** — Confirm rollback is necessary
2. **Notify** — Alert stakeholders of rollback
3. **Halt** — Stop any downstream systems consuming target
4. **Execute** — Run rollback with chosen strategy
5. **Verify** — Confirm rollback completed successfully
6. **Resume** — Re-enable downstream systems
7. **Investigate** — Determine root cause before re-attempting
8. **Document** — Record rollback reason and outcome

→ [Checkpointing](checkpointing.md)
→ [Cutover Strategies](cutover-strategies.md)
→ [Runbooks](runbooks.md)
