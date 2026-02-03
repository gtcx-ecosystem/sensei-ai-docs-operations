# Monitoring

## Real-Time Observability

Sensei provides comprehensive monitoring through metrics, logs, traces, and dashboards. This page covers how to observe what's happening during migrations.

---

### Monitoring Stack

```text
┌─────────────────────────────────────────────────────────────────┐
│                     SENSEI PLATFORM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Agents → OpenTelemetry → ┬→ Metrics (Prometheus)               │
│                           ├→ Logs (Loki/CloudWatch)             │
│                           └→ Traces (Jaeger/X-Ray)              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   YOUR OBSERVABILITY STACK                       │
├─────────────────────────────────────────────────────────────────┤
│  Grafana │ Datadog │ New Relic │ CloudWatch │ Custom            │
└─────────────────────────────────────────────────────────────────┘
```

---

### Key Metrics

#### Migration Progress

| Metric                                     | Type    | Description                |
| ------------------------------------------ | ------- | -------------------------- |
| `sensei_migration_progress_percent`        | Gauge   | Overall completion (0-100) |
| `sensei_migration_records_processed_total` | Counter | Records migrated           |
| `sensei_migration_records_per_second`      | Gauge   | Current throughput         |
| `sensei_migration_phase_current`           | Gauge   | Active phase number        |
| `sensei_migration_eta_seconds`             | Gauge   | Estimated time remaining   |

#### Error Tracking

| Metric                          | Type    | Description                   |
| ------------------------------- | ------- | ----------------------------- |
| `sensei_errors_total`           | Counter | All errors (by type)          |
| `sensei_errors_recovered_total` | Counter | Auto-recovered errors         |
| `sensei_errors_escalated_total` | Counter | Errors requiring human review |
| `sensei_error_rate`             | Gauge   | Errors per 1000 records       |

#### Performance

| Metric                              | Type      | Description             |
| ----------------------------------- | --------- | ----------------------- |
| `sensei_batch_duration_seconds`     | Histogram | Time per batch          |
| `sensei_transform_duration_seconds` | Histogram | Transformation time     |
| `sensei_write_duration_seconds`     | Histogram | Write latency           |
| `sensei_worker_utilization`         | Gauge     | Worker CPU/memory usage |

#### Agent Health

| Metric                             | Type  | Description                       |
| ---------------------------------- | ----- | --------------------------------- |
| `sensei_agent_count`               | Gauge | Active agents by type             |
| `sensei_agent_messages_per_second` | Gauge | Swarm communication rate          |
| `sensei_pattern_library_size`      | Gauge | Patterns in knowledge base        |
| `sensei_pattern_cache_hit_rate`    | Gauge | Local pattern cache effectiveness |

---

### Prometheus Integration

Sensei exposes a Prometheus-compatible metrics endpoint:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'sensei'
    static_configs:
      - targets: ['sensei-api:9090']
    metrics_path: /metrics
    scheme: https
    bearer_token_file: /etc/prometheus/sensei-token
```

Or use the Prometheus remote write endpoint:

```yaml
# Sensei sends metrics to your Prometheus
monitoring:
  prometheus:
    remote_write:
      url: https://prometheus.example.com/api/v1/write
      bearer_token: ${PROMETHEUS_TOKEN}
```

---

### Grafana Dashboards

Sensei provides pre-built Grafana dashboards:

#### Migration Overview Dashboard

- Overall progress across all active migrations
- Throughput trends
- Error rate trends
- Resource utilization

#### Migration Detail Dashboard

- Per-migration deep dive
- Phase progress
- Per-table metrics
- Error breakdown

#### Agent Health Dashboard

- Agent counts and status
- Communication latency
- Pattern library growth
- Memory/CPU per agent type

**Import dashboards:**

```bash
curl https://api.sensei.ai/v1/monitoring/grafana-dashboards \
  -H "Authorization: Bearer sk_live_abc123" \
  -o sensei-dashboards.json

# Import into Grafana
```

---

### Logging

#### Log Format

Structured JSON logs with correlation IDs:

```json
{
  "timestamp": "2026-02-02T14:23:01.234Z",
  "level": "info",
  "component": "worker_agent",
  "agent_id": "worker_47",
  "migration_id": "mig_abc123",
  "correlation_id": "corr_xyz789",
  "message": "Batch completed",
  "data": {
    "table": "orders",
    "batch_number": 1547,
    "records_processed": 10000,
    "errors": 0,
    "duration_ms": 2340
  }
}
```

#### Log Levels

| Level   | When Used                                |
| ------- | ---------------------------------------- |
| `debug` | Detailed execution info (verbose)        |
| `info`  | Normal operational events                |
| `warn`  | Recoverable issues, degraded performance |
| `error` | Failures requiring attention             |

#### Log Shipping

Configure log destination:

```yaml
logging:
  level: info
  format: json

  destinations:
    - type: cloudwatch
      log_group: /sensei/migrations
      region: us-east-1

    - type: loki
      url: https://loki.example.com/loki/api/v1/push

    - type: datadog
      api_key: ${DD_API_KEY}
      site: datadoghq.com
```

---

### Distributed Tracing

Traces span the entire migration pipeline:

```text
Trace: migration_mig_abc123
├── [2.3s] Scout.analyze_schema
│   ├── [0.5s] extract_metadata
│   └── [1.8s] compute_fingerprints
├── [1.2s] Architect.generate_plan
│   ├── [0.3s] query_pattern_library
│   ├── [0.7s] llm_inference
│   └── [0.2s] generate_code
├── [14523s] Worker.execute_migration
│   ├── [0.8s] read_batch
│   ├── [0.4s] transform_batch
│   ├── [0.2s] validate_batch
│   └── [1.1s] write_batch
└── [3600s] KORA.validate
    ├── [60s] structural_validation
    ├── [300s] statistical_validation
    └── [3240s] behavioral_validation
```

#### Trace Export

```yaml
tracing:
  enabled: true
  sample_rate: 0.1 # 10% sampling

  exporter:
    type: jaeger # or zipkin, otlp, xray
    endpoint: https://jaeger.example.com/api/traces
```

---

### Real-Time Streaming

Stream events to your systems:

```python
# Server-Sent Events
import requests

response = requests.get(
    f"https://api.sensei.ai/v1/migrations/{migration_id}/events",
    headers={"Authorization": f"Bearer {api_key}"},
    stream=True
)

for line in response.iter_lines():
    if line:
        event = json.loads(line.decode())
        process_event(event)
```

Event types:

- `progress` — Periodic progress updates
- `phase.started` / `phase.completed` — Phase transitions
- `error.occurred` / `error.recovered` — Error events
- `checkpoint.created` — Checkpoint events

→ [Alerting](alerting.md)
→ [Performance Tuning](performance-tuning.md)
→ [Operations Overview](README.md)
