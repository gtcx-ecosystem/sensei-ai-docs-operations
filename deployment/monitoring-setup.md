# Monitoring Setup

## Observability Infrastructure

Setting up proper monitoring ensures you can observe system health, troubleshoot issues, and optimize performance. This page covers deploying monitoring infrastructure for Hybrid and Self-Hosted deployments.

---

### Monitoring Stack

```text
┌─────────────────────────────────────────────────────────────────────┐
│                       SENSEI PLATFORM                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │   API    │  │  Workers │  │   MABA   │  │   KORA   │            │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘            │
│       │             │             │             │                    │
│       └─────────────┴─────────────┴─────────────┘                    │
│                           │ OpenTelemetry                            │
└───────────────────────────┼──────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   OBSERVABILITY PLATFORM                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Prometheus   │  │    Loki      │  │   Jaeger     │              │
│  │ (Metrics)    │  │   (Logs)     │  │  (Traces)    │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                 │                        │
│         └─────────────────┴─────────────────┘                        │
│                           │                                          │
│                    ┌──────┴──────┐                                  │
│                    │   Grafana   │                                  │
│                    │ (Dashboards)│                                  │
│                    └─────────────┘                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Quick Start (Bundled)

Sensei can deploy a complete monitoring stack:

```yaml
# values.yaml
monitoring:
  enabled: true

  prometheus:
    enabled: true
    retention: 15d
    storage: 50Gi

  loki:
    enabled: true
    retention: 7d
    storage: 100Gi

  jaeger:
    enabled: true
    retention: 7d
    storage: 50Gi

  grafana:
    enabled: true
    adminPassword: ${GRAFANA_PASSWORD}
    dashboards:
      enabled: true # Install Sensei dashboards
```

```bash
helm upgrade sensei sensei/sensei -f values.yaml
```

---

### Prometheus Setup

#### Using Bundled Prometheus

Prometheus is included and pre-configured:

```bash
# Access Prometheus UI
kubectl port-forward svc/prometheus -n sensei 9090:9090
# Open http://localhost:9090
```

#### Using Existing Prometheus

Configure Sensei to expose metrics to your Prometheus:

```yaml
monitoring:
  prometheus:
    enabled: false # Don't deploy bundled

  serviceMonitor:
    enabled: true
    namespace: monitoring # Your Prometheus namespace
    labels:
      prometheus: main
```

**ServiceMonitor:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sensei
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: sensei
  namespaceSelector:
    matchNames:
      - sensei
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

---

### Grafana Dashboards

#### Pre-Built Dashboards

Sensei includes ready-to-use dashboards:

| Dashboard          | Purpose                             |
| ------------------ | ----------------------------------- |
| Sensei Overview    | Platform health at a glance         |
| Migration Status   | Active migrations, progress, errors |
| Worker Performance | Throughput, latency, utilization    |
| Agent Health       | Agent status, communication         |
| Database Health    | PostgreSQL, Redis metrics           |
| Resource Usage     | CPU, memory, network                |

#### Installing Dashboards

```bash
# If using bundled Grafana
# Dashboards are auto-installed

# If using external Grafana
curl https://api.sensei.ai/v1/monitoring/dashboards | \
  jq '.dashboards[]' | \
  while read dashboard; do
    curl -X POST http://grafana:3000/api/dashboards/db \
      -H "Content-Type: application/json" \
      -d "$dashboard"
  done
```

#### Accessing Grafana

```bash
# Port forward
kubectl port-forward svc/grafana -n sensei 3000:3000

# Get admin password
kubectl get secret grafana -n sensei -o jsonpath="{.data.admin-password}" | base64 -d

# Open http://localhost:3000
```

---

### Log Aggregation

#### Using Bundled Loki

```yaml
monitoring:
  loki:
    enabled: true
    storage:
      type: s3
      bucket: sensei-logs
```

#### Using External Log System

**CloudWatch:**

```yaml
logging:
  driver: cloudwatch
  config:
    logGroup: /sensei/logs
    region: us-east-1
```

**Datadog:**

```yaml
logging:
  driver: datadog
  config:
    apiKey: ${DD_API_KEY}
    site: datadoghq.com
```

**Splunk:**

```yaml
logging:
  driver: splunk
  config:
    endpoint: https://splunk.example.com:8088
    token: ${SPLUNK_HEC_TOKEN}
```

---

### Distributed Tracing

#### Using Bundled Jaeger

```bash
# Access Jaeger UI
kubectl port-forward svc/jaeger-query -n sensei 16686:16686
# Open http://localhost:16686
```

#### Using External Tracing

**Datadog APM:**

```yaml
tracing:
  enabled: true
  exporter: datadog
  config:
    apiKey: ${DD_API_KEY}
```

**AWS X-Ray:**

```yaml
tracing:
  enabled: true
  exporter: xray
  config:
    region: us-east-1
```

**Jaeger (external):**

```yaml
tracing:
  enabled: true
  exporter: jaeger
  config:
    endpoint: http://jaeger-collector.monitoring:14268/api/traces
```

---

### Alert Manager

Configure alerting:

```yaml
alertmanager:
  enabled: true

  config:
    receivers:
      - name: 'slack'
        slack_configs:
          - channel: '#sensei-alerts'
            api_url: ${SLACK_WEBHOOK_URL}
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ .Annotations.description }}'

      - name: 'pagerduty'
        pagerduty_configs:
          - routing_key: ${PAGERDUTY_KEY}

    route:
      receiver: 'slack'
      routes:
        - match:
            severity: critical
          receiver: 'pagerduty'
```

---

### Essential Alerts

```yaml
# PrometheusRule for Sensei alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: sensei-alerts
  namespace: sensei
spec:
  groups:
    - name: sensei
      rules:
        - alert: SenseiHighErrorRate
          expr: |
            rate(sensei_errors_total[5m]) / rate(sensei_records_processed_total[5m]) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            description: Error rate above 5% for 5 minutes

        - alert: SenseiMigrationStalled
          expr: |
            increase(sensei_records_processed_total[15m]) == 0 
            and sensei_migration_status == 1
          for: 15m
          labels:
            severity: critical
          annotations:
            description: Migration has not processed records for 15 minutes

        - alert: SenseiDatabaseConnectionFailed
          expr: pg_up{job="sensei-postgresql"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            description: PostgreSQL connection lost

        - alert: SenseiWorkerHighMemory
          expr: |
            container_memory_usage_bytes{container="worker"} / 
            container_spec_memory_limit_bytes{container="worker"} > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            description: Worker memory usage above 90%
```

---

### Verification

After setup, verify monitoring:

```bash
# Check Prometheus targets
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].health'

# Check metrics are being collected
curl http://localhost:9090/api/v1/query?query=sensei_migration_progress_percent

# Check logs in Grafana Explore
# Run query: {app="sensei-api"}

# Check traces in Jaeger
# Search for service: sensei-api
```

→ [Monitoring](../migrations/monitoring.md)
→ [Alerting](../migrations/alerting.md)
→ [Deployment Overview](README.md)
