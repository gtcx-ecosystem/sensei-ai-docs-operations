---
description: Deploy, operate, and maintain Sensei in production
---

# Operations

## Migration Operations

| Phase | Guide |
| --- | --- |
| [Planning](migrations/planning-phase.md) | Scheduling, resource estimation, dependency analysis |
| [Execution](migrations/execution-phase.md) | Checkpointing, incremental runs, parallel migrations |
| [Validation](migrations/validation-phase.md) | Monitoring, alerting, performance tuning |
| [Cutover](migrations/cutover-strategies.md) | Zero-downtime strategies and rollback procedures |
| [Post-Migration](migrations/post-migration.md) | Cleanup, verification, and handoff |
| [Runbooks](migrations/runbooks.md) | Step-by-step guides for common scenarios |

---

## Deployment

| Model | Description |
| --- | --- |
| [SaaS](deployment/saas.md) | Fully managed — nothing to deploy |
| [Hybrid](deployment/hybrid.md) | Data plane in your VPC, control plane in Sensei Cloud |
| [Self-Hosted](deployment/self-hosted.md) | Full control in your own infrastructure |
| [Air-Gapped](deployment/air-gapped.md) | No internet connectivity required |

**Infrastructure:** [Kubernetes](deployment/kubernetes.md) · [Docker](deployment/docker.md) · [Networking](deployment/network.md) · [HA](deployment/high-availability.md) · [Scaling](deployment/scaling.md) · [Security](deployment/security.md) · [Monitoring](deployment/monitoring-setup.md) · [Backup](deployment/backup-recovery.md)

---

## Checklists

- [Migration Readiness](checklists/migration-checklist.md) — pre-flight checks before starting
- [Security Hardening](checklists/security-checklist.md) — production security baseline
- [Compliance Mapping](checklists/compliance-mapping.md) — regulatory requirement coverage
