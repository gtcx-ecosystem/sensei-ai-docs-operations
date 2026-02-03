# Alerting

## Notification Configuration

Sensei alerts you to important events through multiple channels. Configure alerts to match your operational workflow — get notified of what matters, when it matters, through the channels you use.

---

### Alert Types

| Alert Type | Severity | Default Behavior |
|------------|----------|------------------|
| `migration.started` | Info | Notify |
| `migration.completed` | Info | Notify |
| `migration.failed` | Critical | Notify + Page |
| `phase.completed` | Info | Notify |
| `error.escalated` | High | Notify |
| `approval.required` | Medium | Notify |
| `validation.failed` | High | Notify |
| `certificate.ready` | Info | Notify |
| `performance.degraded` | Medium | Notify |
| `resource.exhausted` | Critical | Notify + Page |

---

### Notification Channels

#### Email
```yaml
notifications:
  channels:
    - type: email
      name: team_email
      addresses:
        - team@example.com
        - oncall@example.com
      
      # Optional: per-severity routing
      routing:
        critical: [oncall@example.com, manager@example.com]
        high: [oncall@example.com]
        info: [team@example.com]
```

#### Slack
```yaml
notifications:
  channels:
    - type: slack
      name: slack_alerts
      webhook_url: ${SLACK_WEBHOOK_URL}
      channel: "#migration-alerts"
      
      # Optional: different channels by severity
      routing:
        critical: "#incidents"
        high: "#migration-alerts"
        info: "#migration-updates"
```

#### PagerDuty
```yaml
notifications:
  channels:
    - type: pagerduty
      name: pagerduty
      routing_key: ${PAGERDUTY_ROUTING_KEY}
      
      # Only page for critical alerts
      filter:
        severities: [critical]
```

#### Microsoft Teams
```yaml
notifications:
  channels:
    - type: teams
      name: teams_channel
      webhook_url: ${TEAMS_WEBHOOK_URL}
```

#### Custom Webhook
```yaml
notifications:
  channels:
    - type: webhook
      name: custom_system
      url: https://your-system.com/alerts
      headers:
        Authorization: "Bearer ${CUSTOM_TOKEN}"
      method: POST
```

---

### Alert Rules

Define custom alert rules:

```yaml
alerts:
  rules:
    # Alert if error rate exceeds threshold
    - name: high_error_rate
      condition: error_rate > 0.05
      duration: 5m
      severity: high
      message: "Error rate exceeded 5% for 5 minutes"
      channels: [slack_alerts, pagerduty]
    
    # Alert if throughput drops
    - name: low_throughput
      condition: throughput_per_hour < expected_throughput * 0.5
      duration: 10m
      severity: medium
      message: "Throughput dropped below 50% of expected"
      channels: [slack_alerts]
    
    # Alert on specific error types
    - name: data_loss_error
      condition: error_type == "truncation" AND affected_records > 100
      severity: critical
      message: "Potential data loss: {{ affected_records }} truncation errors"
      channels: [slack_alerts, pagerduty, email]
    
    # Alert on phase duration
    - name: phase_overrun
      condition: phase_duration > phase_estimate * 2
      severity: medium
      message: "Phase {{ phase_name }} running 2x longer than estimated"
      channels: [slack_alerts]
```

---

### Stakeholder Routing

Route alerts based on stakeholder role:

```yaml
stakeholders:
  - role: technical_operator
    channels: [slack_alerts, email]
    alerts: [all]
    
  - role: business_stakeholder
    channels: [email]
    alerts:
      - migration.completed
      - migration.failed
      - phase.completed
      - validation.failed
    
  - role: compliance_officer
    channels: [email]
    alerts:
      - validation.completed
      - validation.failed
      - certificate.ready
    
  - role: executive_sponsor
    channels: [email]
    alerts:
      - migration.completed
      - migration.failed
```

---

### Alert Suppression

Prevent alert fatigue:

```yaml
alerts:
  suppression:
    # Don't repeat same alert within window
    deduplication_window: 15m
    
    # Aggregate similar alerts
    aggregation:
      enabled: true
      window: 5m
      max_per_window: 10
    
    # Suppress during maintenance windows
    maintenance_windows:
      - name: weekly_maintenance
        schedule: "0 2 * * SUN"  # Sundays at 2 AM
        duration: 4h
    
    # Suppress low-severity during off-hours
    off_hours:
      timezone: America/New_York
      suppress_below: high
      hours:
        start: 22:00
        end: 06:00
```

---

### Alert Testing

Test your alert configuration:

```bash
# Send test alert
curl -X POST https://api.sensei.ai/v1/alerts/test \
  -H "Authorization: Bearer sk_live_abc123" \
  -d '{
    "alert_type": "migration.failed",
    "severity": "critical",
    "test_channels": ["slack_alerts", "pagerduty"]
  }'
```

---

### Alert History

View alert history:

```bash
# Get recent alerts
curl https://api.sensei.ai/v1/alerts?limit=50 \
  -H "Authorization: Bearer sk_live_abc123"
```

Response:
```json
{
  "alerts": [
    {
      "id": "alert_abc123",
      "type": "error.escalated",
      "severity": "high",
      "message": "13 errors require human review",
      "migration_id": "mig_xyz789",
      "timestamp": "2026-02-02T14:23:00Z",
      "delivered_to": ["slack_alerts"],
      "acknowledged": true,
      "acknowledged_by": "user@example.com",
      "acknowledged_at": "2026-02-02T14:25:00Z"
    }
  ]
}
```

---

### Alert Acknowledgment

Acknowledge alerts to stop escalation:

```bash
curl -X POST https://api.sensei.ai/v1/alerts/{alert_id}/acknowledge \
  -H "Authorization: Bearer sk_live_abc123" \
  -d '{"comment": "Investigating - will update in 30 minutes"}'
```

Acknowledgment:
- Stops repeat notifications for that alert
- Records who acknowledged and when
- Allows adding investigation notes

→ [Monitoring](monitoring.md)
→ [Error Handling](error-handling.md)
→ [Operations Overview](README.md)
