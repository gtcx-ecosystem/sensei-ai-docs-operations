# Kubernetes Deployment

## Helm Charts, Operators, and Manifests

Sensei runs natively on Kubernetes. This page covers installation and configuration using Helm.

---

### Prerequisites

- Kubernetes 1.25+
- Helm 3.10+
- kubectl configured
- Storage class with dynamic provisioning
- (Optional) GPU nodes for local LLM

---

### Quick Start

```bash
# Add Sensei Helm repository
helm repo add sensei https://charts.sensei.ai
helm repo update

# Install Sensei (minimal configuration)
helm install sensei sensei/sensei \
  --namespace sensei \
  --create-namespace \
  --set license.key=${SENSEI_LICENSE_KEY}
```

---

### Helm Charts

| Chart                 | Description             | Use Case            |
| --------------------- | ----------------------- | ------------------- |
| `sensei/sensei`       | Full platform           | Self-hosted         |
| `sensei/data-plane`   | Data plane only         | Hybrid              |
| `sensei/dependencies` | PostgreSQL, Redis, etc. | Development/testing |

---

### Configuration

**values.yaml:**

```yaml
# Global settings
global:
  imageRegistry: registry.sensei.ai
  imagePullSecrets:
    - name: sensei-registry
  storageClass: gp3

# License
license:
  key: 'your-license-key'

# Control Plane
controlPlane:
  api:
    replicas: 2
    resources:
      requests:
        cpu: 2
        memory: 4Gi
      limits:
        cpu: 4
        memory: 8Gi

  dashboard:
    replicas: 2
    ingress:
      enabled: true
      host: sensei.example.com
      tls:
        enabled: true
        secretName: sensei-tls

  orchestrator:
    replicas: 2
    resources:
      requests:
        cpu: 4
        memory: 8Gi

# Data Plane
dataPlane:
  maba:
    replicas: 2
    resources:
      requests:
        cpu: 8
        memory: 16Gi

  kora:
    replicas: 2
    resources:
      requests:
        cpu: 4
        memory: 8Gi

  amani:
    replicas: 2
    resources:
      requests:
        cpu: 2
        memory: 4Gi

  workers:
    replicas: 4
    autoscaling:
      enabled: true
      minReplicas: 4
      maxReplicas: 32
      targetCPUUtilization: 70
    resources:
      requests:
        cpu: 4
        memory: 8Gi

# Agents
agents:
  scout:
    replicas: 2
  architect:
    replicas: 2
  validator:
    replicas: 2
  meta:
    replicas: 1

# LLM Configuration
llm:
  provider: anthropic # anthropic, openai, local
  apiKey:
    secretName: llm-credentials
    key: api-key
  # For local LLM:
  # provider: local
  # model: llama-3-70b
  # gpu:
  #   enabled: true
  #   count: 2

# Dependencies (set external.enabled=true to use external)
postgresql:
  enabled: true
  auth:
    postgresPassword: ${POSTGRES_PASSWORD}
  primary:
    persistence:
      size: 100Gi

redis:
  enabled: true
  auth:
    password: ${REDIS_PASSWORD}
  master:
    persistence:
      size: 20Gi

vectordb:
  enabled: true
  persistence:
    size: 50Gi

minio:
  enabled: true
  persistence:
    size: 500Gi

# Monitoring
monitoring:
  prometheus:
    enabled: true
    serviceMonitor:
      enabled: true
  grafana:
    enabled: true
    dashboards:
      enabled: true
```

---

### Installation Commands

#### Development Environment

```bash
helm install sensei sensei/sensei \
  --namespace sensei \
  --create-namespace \
  -f values-dev.yaml
```

#### Production Environment

```bash
# Create namespace with resource quotas
kubectl create namespace sensei
kubectl apply -f resource-quota.yaml

# Create secrets first
kubectl create secret generic sensei-license \
  --from-literal=key=${LICENSE_KEY} \
  -n sensei

kubectl create secret generic llm-credentials \
  --from-literal=api-key=${ANTHROPIC_API_KEY} \
  -n sensei

# Install with production values
helm install sensei sensei/sensei \
  --namespace sensei \
  -f values-prod.yaml \
  --wait
```

---

### Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: sensei-quota
  namespace: sensei
spec:
  hard:
    requests.cpu: '200'
    requests.memory: 500Gi
    limits.cpu: '400'
    limits.memory: 1000Gi
    persistentvolumeclaims: '20'
    pods: '100'
```

---

### Pod Disruption Budgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: sensei-api-pdb
  namespace: sensei
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: sensei-api
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: sensei-workers-pdb
  namespace: sensei
spec:
  maxUnavailable: 25%
  selector:
    matchLabels:
      app: sensei-worker
```

---

### Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sensei-default-deny
  namespace: sensei
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sensei-allow-internal
  namespace: sensei
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: sensei
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: sensei
```

---

### Verification

```bash
# Check all pods are running
kubectl get pods -n sensei

# Check services
kubectl get svc -n sensei

# Check ingress
kubectl get ingress -n sensei

# View logs
kubectl logs -l app=sensei-api -n sensei --tail=100

# Test API
kubectl port-forward svc/sensei-api 8080:80 -n sensei
curl http://localhost:8080/health
```

---

### Troubleshooting

#### Pods stuck in Pending

```bash
# Check events
kubectl describe pod <pod-name> -n sensei

# Common causes:
# - Insufficient resources (scale nodes)
# - PVC not bound (check storage class)
# - Image pull error (check secrets)
```

#### Database connection errors

```bash
# Check PostgreSQL is running
kubectl get pods -l app=postgresql -n sensei

# Check service endpoint
kubectl get endpoints postgresql -n sensei

# Test connectivity
kubectl exec -it sensei-api-0 -n sensei -- \
  pg_isready -h postgresql -p 5432
```

→ [Docker](docker.md)
→ [High Availability](high-availability.md)
→ [Scaling](scaling.md)
