# Network Configuration

## Network Configuration

Proper network configuration ensures secure, reliable connectivity between Sensei components and your databases.

---

### Network Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                         YOUR NETWORK                                 │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Private Subnet (Sensei)                                     │   │
│  │  CIDR: 10.0.1.0/24                                          │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │ Control │ │  Data   │ │  Data   │ │  Data   │           │   │
│  │  │  Plane  │ │ Store   │ │  Plane  │ │  Plane  │           │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘           │   │
│  │       └───────────┴───────────┴───────────┘                 │   │
│  │                         │                                    │   │
│  └─────────────────────────┼────────────────────────────────────┘   │
│                            │ Internal routing                       │
│  ┌─────────────────────────┼────────────────────────────────────┐   │
│  │  Private Subnet (Databases)                                  │   │
│  │  CIDR: 10.0.2.0/24      │                                   │   │
│  │       ┌─────────────────┴─────────────────┐                 │   │
│  │  ┌────┴────┐                         ┌────┴────┐            │   │
│  │  │ Source  │                         │ Target  │            │   │
│  │  │   DB    │                         │   DB    │            │   │
│  │  └─────────┘                         └─────────┘            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Firewall Rules

#### Inbound (to Sensei)

| Source       | Destination | Port | Protocol | Purpose           |
| ------------ | ----------- | ---- | -------- | ----------------- |
| Users        | Dashboard   | 443  | HTTPS    | Web UI access     |
| Applications | API Server  | 443  | HTTPS    | API access        |
| Webhooks     | API Server  | 443  | HTTPS    | Webhook callbacks |

#### Internal (Sensei ↔ Sensei)

| Source     | Destination | Port | Protocol | Purpose         |
| ---------- | ----------- | ---- | -------- | --------------- |
| All Sensei | PostgreSQL  | 5432 | TCP      | Metadata        |
| All Sensei | Redis       | 6379 | TCP      | Cache, pub/sub  |
| All Sensei | Vector DB   | 8080 | HTTP     | Pattern library |
| Workers    | MABA        | 8081 | gRPC     | Transformation  |
| Workers    | KORA        | 8082 | gRPC     | Validation      |
| Control    | Data Plane  | 8083 | gRPC     | Orchestration   |

#### Outbound (from Sensei)

| Source     | Destination    | Port   | Protocol | Purpose      |
| ---------- | -------------- | ------ | -------- | ------------ |
| Data Plane | Source DB      | varies | TCP      | Read data    |
| Data Plane | Target DB      | varies | TCP      | Write data   |
| Control    | Sensei Cloud\* | 443    | HTTPS    | Hybrid only  |
| All\*      | LLM API        | 443    | HTTPS    | AI inference |

\*Not required for air-gapped self-hosted

---

### VPC Configuration

#### AWS VPC Example

```hcl
# Terraform example
resource "aws_vpc" "sensei" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "sensei-vpc"
  }
}

# Private subnet for Sensei
resource "aws_subnet" "sensei_private" {
  vpc_id            = aws_vpc.sensei.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "sensei-private"
  }
}

# Security group for Sensei
resource "aws_security_group" "sensei" {
  name        = "sensei-sg"
  vpc_id      = aws_vpc.sensei.id

  # Internal communication
  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    self        = true
  }

  # Dashboard access
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]  # Internal only
  }

  # All outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

### VPC Peering

Connect Sensei VPC to database VPCs:

```text
┌─────────────────┐         ┌─────────────────┐
│   Sensei VPC    │◄───────►│  Database VPC   │
│  10.0.0.0/16    │ Peering │  172.16.0.0/16  │
└─────────────────┘         └─────────────────┘
```

**Steps:**

1. Create peering connection
2. Accept peering request
3. Update route tables in both VPCs
4. Update security groups to allow traffic

---

### Private Link / PrivateLink

For SaaS/Hybrid, use PrivateLink instead of public internet:

**AWS PrivateLink:**

```hcl
resource "aws_vpc_endpoint" "sensei" {
  vpc_id            = aws_vpc.yours.id
  service_name      = "com.amazonaws.vpce.us-east-1.sensei"
  vpc_endpoint_type = "Interface"
  subnet_ids        = [aws_subnet.private.id]
  security_group_ids = [aws_security_group.sensei_endpoint.id]
}
```

**Benefits:**

- Traffic stays on AWS backbone
- No internet gateway required
- Enhanced security

---

### DNS Configuration

#### Internal DNS

```yaml
# CoreDNS ConfigMap example
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        forward . /etc/resolv.conf
        cache 30
    }
    sensei.internal {
        file /etc/coredns/sensei.db
    }
```

#### Service Discovery

Sensei uses Kubernetes DNS by default:

- `sensei-api.sensei.svc.cluster.local`
- `sensei-redis.sensei.svc.cluster.local`
- `sensei-postgresql.sensei.svc.cluster.local`

---

### Load Balancer Configuration

#### Ingress Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sensei-ingress
  namespace: sensei
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: 'true'
spec:
  tls:
    - hosts:
        - sensei.example.com
      secretName: sensei-tls
  rules:
    - host: sensei.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sensei-dashboard
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: sensei-api
                port:
                  number: 80
```

---

### Troubleshooting

#### Connection Timeouts

```bash
# Test connectivity from Sensei to database
kubectl exec -it sensei-worker-0 -n sensei -- \
  nc -zv source-db.example.com 5432

# Check DNS resolution
kubectl exec -it sensei-worker-0 -n sensei -- \
  nslookup source-db.example.com
```

#### Firewall Issues

```bash
# Check if port is reachable
kubectl exec -it sensei-worker-0 -n sensei -- \
  curl -v telnet://source-db.example.com:5432
```

→ [Security](security.md)
→ [Infrastructure Requirements](infrastructure.md)
→ [Deployment Overview](README.md)
