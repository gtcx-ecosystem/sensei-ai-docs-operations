# Runbooks

## Standard Operational Procedures

Runbooks provide step-by-step procedures for common operational scenarios. Use these as templates and adapt for your specific environment.

---

### Runbook Index

| Scenario                                                | Use When                               |
| ------------------------------------------------------- | -------------------------------------- |
| [Starting a Migration](#starting-a-migration)           | Beginning a new migration              |
| [Handling Escalated Errors](#handling-escalated-errors) | Errors need human review               |
| [Pausing/Resuming](#pauseresuming-a-migration)          | Manual intervention needed             |
| [Emergency Stop](#emergency-stop)                       | Critical issue requires immediate halt |
| [Rollback](#rollback-procedure)                         | Migration must be undone               |
| [Performance Degradation](#performance-degradation)     | Migration running slower than expected |
| [Source/Target Unavailable](#sourcetarget-unavailable)  | Connection issues                      |
| [Cutover Execution](#cutover-execution)                 | Go-live procedure                      |

---

### Starting a Migration {#starting-a-migration}

**Trigger:** New migration project approved

**Steps:**

1. **Verify prerequisites**

   ```bash
   # Test source connectivity
   curl -X POST https://api.sensei.ai/v1/connections/test \
     -d '{"type": "postgresql", "connection_string": "..."}'

   # Test target connectivity
   curl -X POST https://api.sensei.ai/v1/connections/test \
     -d '{"type": "snowflake", "account": "..."}'
   ```

2. **Create migration**

   ```bash
   curl -X POST https://api.sensei.ai/v1/migrations \
     -d '{"name": "...", "source": {...}, "target": {...}}'
   ```

3. **Review and approve plan**
   - Log into dashboard
   - Review mapping suggestions
   - Approve or modify as needed

4. **Configure notifications**
   - Set up Slack/email alerts
   - Verify webhook endpoints

5. **Start execution**

   ```bash
   curl -X POST https://api.sensei.ai/v1/migrations/{id}/start
   ```

6. **Monitor initial progress**
   - Watch for first 10 minutes
   - Verify throughput is reasonable
   - Confirm no immediate errors

**Completion criteria:** Migration running, monitoring confirmed

---

### Handling Escalated Errors {#handling-escalated-errors}

**Trigger:** Alert received for escalated error

**Steps:**

1. **Acknowledge alert**

   ```bash
   curl -X POST https://api.sensei.ai/v1/alerts/{alert_id}/acknowledge
   ```

2. **Review error details**

   ```bash
   curl https://api.sensei.ai/v1/migrations/{id}/errors/{error_id}
   ```

3. **Assess impact**
   - How many records affected?
   - Is migration paused or continuing?
   - What downstream tables are blocked?

4. **Determine resolution**
   | Error Type | Common Resolution |
   |-----------|-------------------|
   | FK violation | Check if parent records exist; re-migrate parents |
   | Data format | Apply custom transformation |
   | Business rule | Consult business owner; skip or fix |

5. **Apply resolution**

   ```bash
   # Skip records
   curl -X POST https://api.sensei.ai/v1/migrations/{id}/errors/{error_id}/resolve \
     -d '{"resolution": "skip", "reason": "..."}'

   # Or retry after fix
   curl -X POST https://api.sensei.ai/v1/migrations/{id}/errors/{error_id}/retry
   ```

6. **Verify resolution**
   - Confirm error queue is cleared
   - Verify migration continues

**Completion criteria:** Error resolved, migration progressing

---

### Pause/Resuming a Migration {#pauseresuming-a-migration}

**Trigger:** Need to temporarily stop migration

**Pause steps:**

1. **Issue pause command**

   ```bash
   curl -X POST https://api.sensei.ai/v1/migrations/{id}/pause
   ```

2. **Wait for confirmation**
   - Current batch will complete
   - Checkpoint will be created
   - Status will change to `paused`

3. **Document reason**
   - Add note to migration
   - Notify stakeholders

**Resume steps:**

1. **Verify ready to resume**
   - Issue blocking pause is resolved
   - Resources available
   - Maintenance window open (if applicable)

2. **Issue resume command**

   ```bash
   curl -X POST https://api.sensei.ai/v1/migrations/{id}/resume
   ```

3. **Monitor resumption**
   - Verify progress continues
   - Check for new errors

**Completion criteria:** Migration running or paused as intended

---

### Emergency Stop {#emergency-stop}

**Trigger:** Critical issue requiring immediate halt

**Steps:**

1. **Stop immediately**

   ```bash
   curl -X POST https://api.sensei.ai/v1/migrations/{id}/stop?immediate=true
   ```

2. **Notify stakeholders**
   - Send emergency notification
   - Page on-call if after hours

3. **Assess damage**
   - What phase/table was in progress?
   - How much data was written?
   - Is target in consistent state?

4. **Determine next action**
   - Resume from checkpoint?
   - Rollback required?
   - Investigation needed?

5. **Document incident**
   - Create incident ticket
   - Record timeline
   - Identify root cause

**Completion criteria:** Migration stopped, stakeholders informed, incident documented

---

### Rollback Procedure {#rollback-procedure}

**Trigger:** Migration must be undone

**Steps:**

1. **Confirm rollback decision**
   - Get stakeholder approval
   - Document reason for rollback

2. **Stop migration if running**

   ```bash
   curl -X POST https://api.sensei.ai/v1/migrations/{id}/stop
   ```

3. **Choose rollback strategy**

   ```bash
   # Option 1: Truncate target (fastest)
   curl -X POST https://api.sensei.ai/v1/migrations/{id}/rollback \
     -d '{"strategy": "truncate"}'

   # Option 2: Point-in-time recovery
   curl -X POST https://api.sensei.ai/v1/migrations/{id}/rollback \
     -d '{"strategy": "point_in_time", "restore_to": "2026-02-02T07:59:00Z"}'
   ```

4. **Verify rollback**

   ```bash
   curl https://api.sensei.ai/v1/migrations/{id}/rollback/status
   ```

5. **Restore source traffic**
   - Update connection strings
   - Restart applications pointing to source

6. **Post-rollback**
   - Investigate root cause
   - Plan remediation
   - Reschedule migration

**Completion criteria:** Target restored, source serving traffic, incident documented

---

### Performance Degradation {#performance-degradation}

**Trigger:** Throughput below expected level

**Steps:**

1. **Check bottleneck analysis**

   ```bash
   curl https://api.sensei.ai/v1/migrations/{id}/performance/bottleneck
   ```

2. **Identify bottleneck**
   | Bottleneck | Symptoms | Actions |
   |-----------|----------|---------|
   | Source read | Source CPU high, read latency high | Reduce parallel readers |
   | Transformation | Worker CPU 100% | Add workers or simplify transforms |
   | Target write | Target CPU/IO high | Increase batch size, defer indexes |
   | Network | High latency between components | Check network path |

3. **Apply mitigation**

   ```bash
   # Example: Reduce parallel readers
   curl -X PATCH https://api.sensei.ai/v1/migrations/{id}/config \
     -d '{"source": {"parallel_readers": 2}}'
   ```

4. **Monitor improvement**
   - Watch throughput trend
   - Verify bottleneck shifts

5. **Adjust estimate if needed**
   - Update stakeholders on new ETA
   - Extend maintenance window if required

**Completion criteria:** Throughput acceptable or issue escalated

---

### Source/Target Unavailable {#sourcetarget-unavailable}

**Trigger:** Connection error alert

**Steps:**

1. **Verify connection issue**

   ```bash
   curl -X POST https://api.sensei.ai/v1/connections/{id}/test
   ```

2. **Check database status**
   - Is database server running?
   - Is there a network issue?
   - Are credentials still valid?

3. **Wait for auto-recovery**
   - Sensei retries with exponential backoff
   - Migration may auto-resume when connection restored

4. **If not auto-recovering**
   - Fix underlying issue
   - Resume migration manually

5. **Post-incident**
   - Determine if data was lost (check checkpoint)
   - Verify migration state is consistent

**Completion criteria:** Connection restored, migration continuing

---

### Cutover Execution {#cutover-execution}

**Trigger:** Migration complete, ready for cutover

**Pre-cutover (T-1 hour):**

- [ ] Verify migration status is `certified`
- [ ] Verify incremental sync lag < 1 minute
- [ ] Notify stakeholders cutover is starting
- [ ] Prepare rollback procedure

**Cutover execution:**

T-0:00 — Announce cutover starting

```bash
# Optional: Pause source writes if needed
```

T-0:05 — Final sync

```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/sync
curl https://api.sensei.ai/v1/migrations/{id}/lag  # Wait for 0
```

T-0:10 — Update application configuration

```bash
# Update connection strings to point to target
# This step is environment-specific
```

T-0:15 — Start applications

```bash
# Start applications pointing to target
# Verify applications starting
```

T-0:20 — Smoke tests

```bash
# Run critical path tests
# Verify core functionality
```

T-0:30 — Announce cutover complete

**Post-cutover monitoring:**

- Watch error rates for 1 hour
- Compare latency to baseline
- Monitor user feedback

**Rollback trigger criteria:**

- Error rate > 5%
- P95 latency > 2x baseline
- Critical functionality broken

**Completion criteria:** Applications running on target, metrics healthy

→ [Operations Overview](README.md)
