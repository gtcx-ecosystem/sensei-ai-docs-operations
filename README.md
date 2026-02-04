---
description: Deploy, operate, and maintain Sensei in production
---

# Operations Home

Sensei migrations move through five phases — planning, execution, validation, cutover, and post-migration. Each phase has built-in checkpointing, so a failure at any stage rolls back cleanly without data loss. This section covers every phase, plus the deployment and infrastructure guides you need to run Sensei in production.

***

## Migration Lifecycle

Every migration follows the same progression. Start with planning, and each phase hands off to the next.

```
Plan → Execute → Validate → Cut Over → Complete
```

<table><thead><tr><th width="144.9765625">Phase</th><th width="409.43359375">What Happens</th><th>Guide</th></tr></thead><tbody><tr><td><strong>Plan</strong></td><td>Estimate resources, map dependencies, set schedule</td><td><a href="migrations/planning-phase.md">Planning Phase</a></td></tr><tr><td><strong>Execute</strong></td><td>Run transforms with checkpointing and parallelism</td><td><a href="migrations/execution-phase.md">Execution Phase</a></td></tr><tr><td><strong>Validate</strong></td><td>Monitor progress, alert on anomalies, tune performance</td><td><a href="migrations/validation-phase.md">Validation Phase</a></td></tr><tr><td><strong>Cut Over</strong></td><td>Switch traffic with zero-downtime strategies</td><td><a href="migrations/cutover-strategies.md">Cutover Strategies</a></td></tr><tr><td><strong>Complete</strong></td><td>Verify results, clean up, hand off to operations</td><td><a href="migrations/post-migration.md">Post-Migration</a></td></tr></tbody></table>

**Deep dives:** [Migration Lifecycle](migrations/migration-lifecycle.md) · [Checkpointing](migrations/checkpointing.md) · [Incremental Migrations](migrations/incremental-migrations.md) · [Parallel Migrations](migrations/parallel-migrations.md) · [Rollback Procedures](migrations/rollback-procedures.md) · [Runbooks](migrations/runbooks.md)

***

## Deployment Models

Choose a deployment model based on your data residency, compliance, and operational requirements.

<table><thead><tr><th width="140.875">Model</th><th width="244.453125">Where It Runs</th><th>Best For</th></tr></thead><tbody><tr><td><a href="deployment/saas.md"><strong>SaaS</strong></a></td><td>Sensei Cloud</td><td>Teams that want zero infrastructure overhead</td></tr><tr><td><a href="deployment/hybrid.md"><strong>Hybrid</strong></a></td><td>Data plane in your VPC, control plane in Sensei Cloud</td><td>Regulated industries needing data residency</td></tr><tr><td><a href="deployment/self-hosted.md"><strong>Self-Hosted</strong></a></td><td>Your infrastructure, fully managed by you</td><td>Full control over updates, networking, and security</td></tr><tr><td><a href="deployment/air-gapped.md"><strong>Air-Gapped</strong></a></td><td>Isolated network, no internet</td><td>Government, defense, classified environments</td></tr></tbody></table>

***

## Infrastructure

Guides for running Sensei reliably at scale, regardless of deployment model.

<table><thead><tr><th width="152.46875">Area</th><th>Guides</th></tr></thead><tbody><tr><td><strong>Containers</strong></td><td><a href="deployment/kubernetes.md">Kubernetes</a> · <a href="deployment/docker.md">Docker</a></td></tr><tr><td><strong>Networking</strong></td><td><a href="deployment/network.md">Network Configuration</a> · <a href="deployment/security.md">Security</a></td></tr><tr><td><strong>Reliability</strong></td><td><a href="deployment/high-availability.md">High Availability</a> · <a href="deployment/scaling.md">Scaling</a> · <a href="deployment/backup-recovery.md">Backup and Recovery</a></td></tr><tr><td><strong>Day-2 Ops</strong></td><td><a href="deployment/configuration.md">Configuration</a> · <a href="deployment/monitoring-setup.md">Monitoring</a> · <a href="deployment/upgrades.md">Upgrades</a></td></tr></tbody></table>

***

## Pre-Flight Checklists

Run these before going live.

<table><thead><tr><th width="315.796875">Checklist</th><th>Purpose</th></tr></thead><tbody><tr><td><a href="checklists/migration-checklist.md"><strong>Migration Readiness</strong></a></td><td>Verify source/target connectivity, schema compatibility, resource capacity</td></tr><tr><td><a href="checklists/security-checklist.md"><strong>Security Hardening</strong></a></td><td>TLS, auth, secrets management, network policies</td></tr><tr><td><a href="checklists/compliance-mapping.md"><strong>Compliance Mapping</strong></a></td><td>Map Sensei controls to SOC 2, HIPAA, GDPR, PCI-DSS requirements</td></tr></tbody></table>
