---
description: Deploy, operate, and maintain Sensei in production
---

# Operations

Sensei migrations move through five phases — planning, execution, validation, cutover, and post-migration. Each phase has built-in checkpointing, so a failure at any stage rolls back cleanly without data loss. This section covers every phase, plus the deployment and infrastructure guides you need to run Sensei in production.

---

## Migration Lifecycle

Every migration follows the same progression. Start with planning, and each phase hands off to the next.

```
Plan → Execute → Validate → Cut Over → Complete
```

| Phase | What Happens | Guide |
| --- | --- | --- |
| **Plan** | Estimate resources, map dependencies, set schedule | [Planning Phase](migrations/planning-phase.md) |
| **Execute** | Run transforms with checkpointing and parallelism | [Execution Phase](migrations/execution-phase.md) |
| **Validate** | Monitor progress, alert on anomalies, tune performance | [Validation Phase](migrations/validation-phase.md) |
| **Cut Over** | Switch traffic with zero-downtime strategies | [Cutover Strategies](migrations/cutover-strategies.md) |
| **Complete** | Verify results, clean up, hand off to operations | [Post-Migration](migrations/post-migration.md) |

**Deep dives:** [Migration Lifecycle](migrations/migration-lifecycle.md) · [Checkpointing](migrations/checkpointing.md) · [Incremental Migrations](migrations/incremental-migrations.md) · [Parallel Migrations](migrations/parallel-migrations.md) · [Rollback Procedures](migrations/rollback-procedures.md) · [Runbooks](migrations/runbooks.md)

---

## Deployment Models

Choose a deployment model based on your data residency, compliance, and operational requirements.

| Model | Where It Runs | Best For |
| --- | --- | --- |
| [**SaaS**](deployment/saas.md) | Sensei Cloud | Teams that want zero infrastructure overhead |
| [**Hybrid**](deployment/hybrid.md) | Data plane in your VPC, control plane in Sensei Cloud | Regulated industries needing data residency |
| [**Self-Hosted**](deployment/self-hosted.md) | Your infrastructure, fully managed by you | Full control over updates, networking, and security |
| [**Air-Gapped**](deployment/air-gapped.md) | Isolated network, no internet | Government, defense, classified environments |

---

## Infrastructure

Guides for running Sensei reliably at scale, regardless of deployment model.

| Area | Guides |
| --- | --- |
| **Containers** | [Kubernetes](deployment/kubernetes.md) · [Docker](deployment/docker.md) |
| **Networking** | [Network Configuration](deployment/network.md) · [Security](deployment/security.md) |
| **Reliability** | [High Availability](deployment/high-availability.md) · [Scaling](deployment/scaling.md) · [Backup and Recovery](deployment/backup-recovery.md) |
| **Day-2 Ops** | [Configuration](deployment/configuration.md) · [Monitoring](deployment/monitoring-setup.md) · [Upgrades](deployment/upgrades.md) |

---

## Pre-Flight Checklists

Run these before going live.

| Checklist | Purpose |
| --- | --- |
| [**Migration Readiness**](checklists/migration-checklist.md) | Verify source/target connectivity, schema compatibility, resource capacity |
| [**Security Hardening**](checklists/security-checklist.md) | TLS, auth, secrets management, network policies |
| [**Compliance Mapping**](checklists/compliance-mapping.md) | Map Sensei controls to SOC 2, HIPAA, GDPR, PCI-DSS requirements |
