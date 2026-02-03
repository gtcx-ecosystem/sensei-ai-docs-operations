# Scheduling

## Migration Windows and Automation

Schedule migrations for optimal timing — during maintenance windows, off-peak hours, or as part of automated pipelines.

---

### Scheduling Options

| Option                 | Use Case                                   |
| ---------------------- | ------------------------------------------ |
| **Immediate**          | Start now, run continuously                |
| **Scheduled Start**    | Begin at specific time                     |
| **Maintenance Window** | Run only during defined windows            |
| **Triggered**          | Start based on external event              |
| **Recurring**          | Run on a schedule (incremental migrations) |

---

### Immediate Execution

```bash
# Start immediately
curl -X POST https://api.sensei.ai/v1/migrations/{id}/start
```

Migration starts within seconds and runs continuously until complete.

---

### Scheduled Start

```bash
# Start at specific time
curl -X POST https://api.sensei.ai/v1/migrations/{id}/schedule \
  -d '{
    "start_at": "2026-02-03T02:00:00Z",
    "timezone": "America/New_York"
  }'
```

Migration will start at the specified time automatically.

**Pre-flight checks:**

- Source/target connectivity verified 30 minutes before
- Resources provisioned 15 minutes before
- Stakeholders notified 10 minutes before

---

### Maintenance Windows

Define windows when migration is allowed to run:

```yaml
schedule:
  maintenance_windows:
    - name: weekend_window
      days: [saturday, sunday]
      start_time: '00:00'
      end_time: '23:59'
      timezone: America/New_York

    - name: nightly_window
      days: [monday, tuesday, wednesday, thursday, friday]
      start_time: '22:00'
      end_time: '06:00'
      timezone: America/New_York

  behavior:
    outside_window: pause # pause | continue | fail
    window_ending_warning: 30m # Warn 30 min before window ends
    complete_current_batch: true
```

**How it works:**

```text
Window opens (Sat 00:00)
        │
        ↓
Migration starts or resumes
        │
        ... running ...
        │
Window ending warning (Sun 23:30)
        │
        ↓
Complete current phase if possible
        │
Window closes (Sun 23:59)
        │
        ↓
Migration pauses, creates checkpoint
        │
        ... waiting ...
        │
Window opens (Mon 22:00)
        │
        ↓
Migration resumes from checkpoint
```

---

### Triggered Execution

Start migrations based on external events:

```yaml
schedule:
  trigger:
    type: webhook
    webhook_url: "https://api.sensei.ai/v1/migrations/{id}/trigger"
    secret: ${TRIGGER_SECRET}

    # Or: API-based trigger
    type: api
    await_approval: false  # Start immediately on trigger
```

**Integration examples:**

**GitHub Actions:**

```yaml
- name: Trigger migration
  run: |
    curl -X POST https://api.sensei.ai/v1/migrations/$MIG_ID/start \
      -H "Authorization: Bearer $SENSEI_API_KEY"
```

**Terraform:**

```hcl
resource "null_resource" "trigger_migration" {
  depends_on = [aws_db_instance.target]

  provisioner "local-exec" {
    command = "curl -X POST https://api.sensei.ai/v1/migrations/${var.migration_id}/start"
  }
}
```

---

### Recurring Schedules

For incremental/continuous migrations:

```yaml
schedule:
  recurring:
    enabled: true
    cron: '0 2 * * *' # Daily at 2 AM
    timezone: America/New_York

    behavior:
      skip_if_running: true
      max_duration: 4h
      notify_on_skip: true
```

See [Incremental Migrations](incremental-migrations.md) for full CDC setup.

---

### Schedule Management

#### View Schedule

```bash
curl https://api.sensei.ai/v1/migrations/{id}/schedule
```

#### Modify Schedule

```bash
curl -X PATCH https://api.sensei.ai/v1/migrations/{id}/schedule \
  -d '{"start_at": "2026-02-04T02:00:00Z"}'
```

#### Cancel Schedule

```bash
curl -X DELETE https://api.sensei.ai/v1/migrations/{id}/schedule
```

---

### Calendar Integration

Export schedules to calendar format:

```bash
# Get iCal feed
curl https://api.sensei.ai/v1/calendar/migrations.ics \
  -H "Authorization: Bearer sk_live_abc123"
```

Subscribe to this URL in Google Calendar, Outlook, etc. to see upcoming migrations.

---

### Conflicts and Dependencies

Define migration dependencies:

```yaml
schedule:
  dependencies:
    # This migration must complete first
    wait_for:
      - migration_id: mig_reference_data
        condition: completed

    # Don't run simultaneously with these
    exclude_concurrent:
      - migration_id: mig_heavy_analytics
        reason: 'Competes for same target resources'
```

Sensei will automatically delay start if dependencies aren't met.

---

### Best Practices

1. **Schedule during low-usage periods** — Less impact on source/target
2. **Allow buffer time** — Don't schedule back-to-back
3. **Consider timezone** — Whose "night" matters?
4. **Test window transitions** — Verify pause/resume works
5. **Set up monitoring** — Watch scheduled jobs

→ [Parallel Migrations](parallel-migrations.md)
→ [Incremental Migrations](incremental-migrations.md)
→ [Operations Overview](README.md)
