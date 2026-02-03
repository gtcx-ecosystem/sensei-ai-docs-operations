# Error Handling

## Error Classification, Recovery, and Escalation

Sensei handles errors autonomously whenever possible. When human intervention is required, the system provides clear context and recommended actions.

---

### Error Classification

| Category          | Examples                             | Auto-Recovery | Typical Rate |
| ----------------- | ------------------------------------ | ------------- | ------------ |
| **Encoding**      | UTF-8 mismatch, character set issues | 95%+          | 0.01-0.1%    |
| **Type Mismatch** | String→Integer, Date format          | 90%+          | 0.1-1%       |
| **Constraint**    | FK violation, NOT NULL, unique       | 70%+          | 0.1-0.5%     |
| **Truncation**    | String too long for target           | 85%+          | 0.01-0.1%    |
| **Network**       | Connection timeout, packet loss      | 99%+          | Varies       |
| **Resource**      | Memory, disk, CPU exhaustion         | 90%+          | Rare         |
| **Logic**         | Business rule violation              | 20%+          | Varies       |

---

### Auto-Recovery Flow

```text
Error Detected
      │
      ↓
┌─────────────────┐
│ Classify Error  │
│ (pattern match) │
└────────┬────────┘
         │
         ↓
┌─────────────────┐     No      ┌─────────────────┐
│ Known Pattern?  │ ──────────→ │ LLM Analysis    │
└────────┬────────┘             └────────┬────────┘
         │ Yes                           │
         ↓                               ↓
┌─────────────────┐             ┌─────────────────┐
│ Apply Known Fix │             │ Generate Fix    │
└────────┬────────┘             └────────┬────────┘
         │                               │
         └──────────┬────────────────────┘
                    ↓
             ┌──────────────┐
             │  Retry       │
             └──────┬───────┘
                    │
         ┌──────────┴──────────┐
         │ Success?            │
         ├─────────────────────┤
         │ Yes        No       │
         ↓            ↓        │
    ┌─────────┐  ┌─────────┐   │
    │ Log &   │  │ Escalate│   │
    │ Continue│  │ to Human│   │
    └─────────┘  └─────────┘   │
```

---

### Common Error Types and Fixes

#### Encoding Errors

```yaml
error:
  type: encoding_mismatch
  source: Latin-1 byte sequence
  target: UTF-8 expected
  sample: "Caf\xe9"

auto_fix:
  strategy: force_encode
  action: "source.encode('latin-1').decode('utf-8', errors='replace')"
  success_rate: 97%
```

#### Type Conversion

```yaml
error:
  type: type_mismatch
  source: VARCHAR "2024-01-15"
  target: DATE

auto_fix:
  strategy: parse_date
  action: "TO_DATE(value, 'YYYY-MM-DD')"
  success_rate: 94%
  fallback: 'NULL with warning'
```

#### FK Violations

```yaml
error:
  type: constraint_violation
  constraint: orders_customer_fk
  message: 'No matching customer_id: 99847'

auto_fix:
  strategy: defer_record
  action: 'Queue record, retry after customers table complete'
  success_rate: 85%
  fallback: 'Escalate if still failing'
```

---

### Escalation

When auto-recovery fails, errors are escalated:

```yaml
escalated_error:
  id: err_abc123
  migration_id: mig_xyz789
  timestamp: '2026-02-02T14:23:00Z'

  error:
    type: constraint_violation
    message: 'Foreign key violation: customer_id 99847 not found'
    table: orders
    record_id: 47291

  context:
    affected_records: 234
    attempted_fixes: 3
    last_fix_attempt: 'Defer and retry - still failing'

  analysis:
    likely_cause: 'Customer records deleted from source after migration started'
    confidence: 0.87

  recommended_actions:
    - 'Check if customer 99847 was deleted from source'
    - 'Manually add customer record to target, then retry'
    - 'Skip records if business confirms customers are obsolete'

  impact:
    blocked_records: 234
    downstream_tables: ['order_items', 'payments']
    business_impact: 'Low - appears to be historical data'
```

---

### Error Queue

Escalated errors are placed in a queue for review:

```bash
# List escalated errors
curl https://api.sensei.ai/v1/migrations/{id}/errors?status=escalated

# Get error details
curl https://api.sensei.ai/v1/migrations/{id}/errors/{error_id}

# Resolve error
curl -X POST https://api.sensei.ai/v1/migrations/{id}/errors/{error_id}/resolve \
  -d '{
    "resolution": "skip",
    "reason": "Business confirmed these are obsolete customers"
  }'

# Retry error
curl -X POST https://api.sensei.ai/v1/migrations/{id}/errors/{error_id}/retry \
  -d '{"fix_applied": "Manually inserted customer 99847"}'
```

---

### Resolution Options

| Resolution    | When to Use                   | Effect                                   |
| ------------- | ----------------------------- | ---------------------------------------- |
| **retry**     | After fixing underlying cause | Re-attempt migration of affected records |
| **skip**      | Records are not needed        | Skip records, log for audit              |
| **default**   | Apply default value           | Replace problematic value with default   |
| **transform** | Apply custom fix              | Apply specified transformation           |
| **escalate**  | Need higher authority         | Move to higher-level review queue        |

---

### Error Thresholds

Configure when errors trigger alerts or pauses:

```yaml
error_handling:
  thresholds:
    # Alert when error rate exceeds
    alert_threshold: 0.01 # 1% error rate

    # Pause migration when error rate exceeds
    pause_threshold: 0.05 # 5% error rate

    # Fail migration when errors exceed
    fail_threshold: 0.10 # 10% error rate

    # Maximum consecutive errors before pause
    max_consecutive: 100

    # Maximum escalated errors before pause
    max_escalated: 50
```

---

### Error Patterns and Learning

Successfully recovered errors improve future migrations:

```yaml
# When an error is successfully recovered
learning_event:
  error_signature: 'encoding:latin1_to_utf8:0xe9'
  solution: "encode('latin-1').decode('utf-8', 'replace')"
  success: true

  # Stored in pattern library
  pattern:
    id: pat_enc_47291
    type: error_recovery
    applies_to: ['encoding_mismatch', 'latin-1', 'utf-8']
    solution_code: "source.encode('latin-1').decode('utf-8', 'replace')"
    success_rate: 0.97

  # Future migrations benefit
  next_migration:
    same_error_occurs: true
    auto_apply_fix: true
    recovery_time: <50ms (vs 200ms LLM inference)
```

→ [Checkpointing](checkpointing.md)
→ [Rollback](rollback.md)
→ [Monitoring](monitoring.md)
