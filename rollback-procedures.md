# Rollback Procedures

## When Things Go Wrong: Recovering Gracefully

Even with Sensei's high success rates, situations arise where rollback is necessary. This guide covers rollback strategies for every migration phase.

---

## Rollback Philosophy

Sensei approaches rollback with three principles:

1. **Prevent, don't just recover:** Most rollback scenarios are preventable with proper validation
2. **Minimize blast radius:** Design migrations so rollback affects the smallest possible scope
3. **Preserve evidence:** Never destroy data that might explain what went wrong

---

## Rollback Decision Matrix

| Situation                          | Recommended Action                     | Time Pressure |
| ---------------------------------- | -------------------------------------- | ------------- |
| Validation failure during testing  | Fix and retry                          | Low           |
| Performance issue during execution | Pause, investigate, resume or rollback | Medium        |
| Data corruption detected           | Immediate rollback                     | High          |
| Business logic error post-cutover  | Assess impact, targeted rollback       | High          |
| External system dependency failure | Pause and wait, or rollback            | Variable      |

---

## Pre-Migration Rollback Preparation

Before any migration begins, ensure rollback readiness:

### 1. Backup Verification

```yaml
pre_migration_checks:
  - type: backup_verification
    source:
      full_backup_exists: required
      backup_tested: required
      recovery_time_objective: 4h
    target:
      schema_only_backup: recommended
      point_in_time_recovery: enabled
```

### 2. Rollback Script Generation

Sensei automatically generates rollback scripts during planning:

```bash
# Generated rollback script
sensei migrations rollback-script mig_abc123 --output rollback.sql
```

The script includes:

- Table drops for migrated objects (if target was empty)
- Constraint restoration
- Sequence reset commands
- Index recreation statements

### 3. Cutover Window Planning

```yaml
cutover_plan:
  window_start: '2026-02-15T02:00:00Z'
  window_end: '2026-02-15T06:00:00Z'

  rollback_decision_point: '2026-02-15T04:00:00Z' # Latest time to decide

  contacts:
    decision_maker: 'cto@example.com'
    technical_lead: 'dba@example.com'
    business_owner: 'ops@example.com'
```

---

## Rollback by Phase

### Phase 1: Planning Rollback

**When:** Migration plan is unsatisfactory before execution begins

**Action:** Simply discard the plan and start over

```bash
# Cancel a migration in planning phase
sensei migrations cancel mig_abc123 --reason "Schema mapping needs revision"

# Create new migration with updated configuration
sensei migrations create --config updated-migration.yaml
```

**Impact:** None — no data has moved yet

---

### Phase 2: Execution Rollback

**When:** Issues detected during data transfer

#### Option A: Pause and Resume

For recoverable issues (network interruption, temporary resource constraint):

```bash
# Pause migration
sensei migrations pause mig_abc123

# Check status and diagnose
sensei migrations status mig_abc123 --detailed

# Resume when ready
sensei migrations resume mig_abc123
```

#### Option B: Partial Rollback

If specific tables or batches have issues:

```bash
# Rollback specific tables
sensei migrations rollback mig_abc123 \
  --tables orders,order_items \
  --keep-progress  # Don't lose progress on other tables
```

#### Option C: Full Rollback

If the migration cannot proceed:

```bash
# Full rollback to pre-migration state
sensei migrations rollback mig_abc123 \
  --full \
  --verify-source  # Confirm source is unchanged
```

**Rollback Verification:**

```yaml
rollback_verification:
  - source_row_counts: unchanged
  - source_data_integrity: verified
  - target_cleaned: true
  - indexes_restored: true
  - sequences_reset: true
```

---

### Phase 3: Validation Rollback

**When:** KORA validation fails after execution completes

This is actually the ideal failure mode — issues are caught before cutover.

```bash
# Check validation failures
sensei validations report mig_abc123 --failures-only

# Detailed divergence analysis
sensei validations divergence mig_abc123 --export report.json
```

**Decision tree:**

```
Validation Failed
├── Data accuracy issue?
│   ├── Transformation bug → Fix and re-execute affected tables
│   └── Source data issue → Clean source and retry
├── Behavioral divergence?
│   ├── Missing stored procedure → Migrate procedure and revalidate
│   └── Different execution plan → Tune target indexes
└── Performance issue?
    └── Optimize and revalidate (don't rollback)
```

---

### Phase 4: Post-Cutover Rollback

**When:** Issues discovered after business users are on the new system

This is the most complex scenario. Options depend on the cutover strategy used:

#### Big Bang Cutover Rollback

```yaml
big_bang_rollback:
  steps: 1. Announce rollback (all-hands communication)
    2. Stop all application access to target
    3. Capture any transactions since cutover
    4. Restore source system access
    5. Apply post-cutover transactions to source (if possible)
    6. Resume business operations on source
    7. Diagnose failure before retry
```

**Data reconciliation:**

```bash
# Export transactions made after cutover
sensei export post-cutover-transactions mig_abc123 \
  --since "2026-02-15T04:00:00Z" \
  --output post-cutover-data.json

# Apply to source (manual review required)
sensei apply-transactions \
  --source prod-source \
  --data post-cutover-data.json \
  --dry-run  # Review before applying
```

#### Parallel Running Rollback

If you maintained parallel operations:

```yaml
parallel_rollback:
  steps: 1. Switch application traffic back to source (typically load balancer)
    2. Continue parallel writes until confident
    3. Investigate target issues
    4. Resume migration when ready
```

This is why parallel running is recommended for critical systems.

#### Phased Cutover Rollback

```yaml
phased_rollback:
  scope: Only roll back affected phase
  steps: 1. Identify affected user groups or data domains
    2. Switch those users/domains back to source
    3. Continue operating hybrid during investigation
    4. Resume migration when fixed
```

---

## Automated Rollback Triggers

Configure Sensei to automatically roll back under specific conditions:

```yaml
auto_rollback:
  enabled: true
  triggers:
    - condition: error_rate > 5%
      action: pause_and_alert

    - condition: row_count_mismatch > 1%
      action: stop_and_investigate

    - condition: validation_failure
      severity: critical
      action: auto_rollback

    - condition: duration > planned_duration * 2
      action: alert_and_assess
```

---

## Rollback Testing

Test your rollback procedures before you need them:

```bash
# Create a test migration in staging
sensei migrations create --config migration.yaml --environment staging

# Execute migration
sensei migrations start mig_staging_123

# Test rollback
sensei migrations rollback mig_staging_123 --full --verify

# Verify rollback succeeded
sensei migrations verify-rollback mig_staging_123
```

**Rollback drill checklist:**

- [ ] Backup restoration tested
- [ ] Rollback script generated and reviewed
- [ ] Rollback executed in staging
- [ ] Source system verified unchanged
- [ ] Target system cleaned
- [ ] Rollback duration measured
- [ ] Communication plan tested

---

## Post-Rollback Analysis

After any rollback, conduct a systematic analysis:

```yaml
post_rollback_analysis:
  questions:
    - What triggered the rollback?
    - When was the issue first detectable?
    - Could validation have caught this earlier?
    - Was the rollback procedure smooth?
    - What changes are needed before retry?

  artifacts_to_preserve:
    - Migration logs
    - Validation reports
    - Performance metrics
    - Error messages
    - Timeline of events
```

---

## Rollback Metrics

Track these metrics to improve rollback readiness:

| Metric                   | Target      | Measurement                                |
| ------------------------ | ----------- | ------------------------------------------ |
| Rollback decision time   | <30 minutes | Time from issue to decision                |
| Rollback execution time  | <4 hours    | Time from decision to completion           |
| Data loss                | Zero        | Transactions lost during rollback          |
| Rollback success rate    | 100%        | Successful rollbacks / Attempted rollbacks |
| Rollback drill frequency | Quarterly   | Tested rollback procedures                 |

→ [Cutover Strategies](cutover-strategies.md)
→ [Error Handling](error-handling.md)
→ [Monitoring](monitoring.md)
