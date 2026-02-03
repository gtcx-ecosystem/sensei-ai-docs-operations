# Overview

## Running Sensei in Your Environment

Sensei offers flexible deployment options — from fully-managed SaaS to air-gapped on-premises installations. This section covers deployment models, infrastructure requirements, security configuration, and operational setup.

---

### Deployment Models

| Model                             | Description                                | Best For                               |
| --------------------------------- | ------------------------------------------ | -------------------------------------- |
| [**SaaS**](saas.md)               | Fully managed by Sensei                    | Most customers, fastest start          |
| [**Hybrid**](hybrid.md)           | Control plane SaaS, data plane in your VPC | Data residency requirements            |
| [**Self-Hosted**](self-hosted.md) | Fully on-premises                          | Air-gapped, high-security environments |

```text
┌─────────────────────────────────────────────────────────────────────┐
│                     DEPLOYMENT SPECTRUM                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SAAS ◄─────────────────────────────────────────────► SELF-HOSTED  │
│                                                                      │
│  ● Fully managed          ● Hybrid               ● Fully controlled │
│  ● Fastest setup          ● Data stays local     ● Air-gapped OK    │
│  ● Auto-updates           ● Some control         ● You manage all   │
│  ● Multi-tenant           ● Your security        ● Single-tenant    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Quick Comparison

| Capability     | SaaS              | Hybrid                  | Self-Hosted        |
| -------------- | ----------------- | ----------------------- | ------------------ |
| Setup time     | Minutes           | Hours                   | Days               |
| Data location  | Sensei cloud      | Your VPC                | Your datacenter    |
| Updates        | Automatic         | Automatic control plane | Manual             |
| Scaling        | Automatic         | Semi-automatic          | Manual             |
| Network access | Internet required | Internet for control    | Air-gapped OK      |
| Compliance     | SOC 2, GDPR       | SOC 2, GDPR, custom     | Your certification |
| Support        | Included          | Included                | Available          |

---

### Deployment Topics

| Topic                                                | Description                             |
| ---------------------------------------------------- | --------------------------------------- |
| [**Infrastructure Requirements**](infrastructure.md) | Compute, network, storage needs         |
| [**Network Configuration**](network.md)              | Firewall rules, VPC setup, connectivity |
| [**Security**](security.md)                          | Encryption, authentication, compliance  |
| [**Kubernetes**](kubernetes.md)                      | Helm charts, operators, manifests       |
| [**Docker**](docker.md)                              | Container images, compose files         |
| [**High Availability**](high-availability.md)        | Redundancy, failover, disaster recovery |
| [**Scaling**](scaling.md)                            | Horizontal and vertical scaling         |
| [**Upgrades**](upgrades.md)                          | Version management, rollback            |
| [**Backup & Recovery**](backup-recovery.md)          | Data protection, restore procedures     |
| [**Monitoring Setup**](monitoring-setup.md)          | Observability infrastructure            |

---

### Getting Started by Deployment Model

#### SaaS (Recommended)

1. Sign up at [sensei.ai](https://sensei.ai)
2. Connect your databases
3. Start migrating

→ [SaaS Deployment](saas.md)

#### Hybrid

1. Create Sensei account
2. Deploy data plane in your VPC
3. Connect data plane to control plane
4. Connect your databases

→ [Hybrid Deployment](hybrid.md)

#### Self-Hosted

1. Obtain license and container images
2. Deploy Kubernetes cluster (or use existing)
3. Install Sensei via Helm
4. Configure and validate

→ [Self-Hosted Deployment](self-hosted.md)

---

### Support

All deployment models include access to:

- Documentation and knowledge base
- Community forums
- Email support (response time varies by plan)
- Enterprise: Dedicated support, professional services

→ [Support Options](../../enterprise/resources/support/)
