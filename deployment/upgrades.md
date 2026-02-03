# Upgrades

## Version Management and Rollback

Keeping Sensei up to date ensures you have the latest features, performance improvements, and security patches. This page covers upgrade procedures for each deployment model.

---

### Version Policy

| Version Type | Format | Frequency | Notes                             |
| ------------ | ------ | --------- | --------------------------------- |
| Major        | X.0.0  | Annually  | May have breaking changes         |
| Minor        | 1.X.0  | Monthly   | New features, backward compatible |
| Patch        | 1.0.X  | As needed | Bug fixes, security patches       |

**Support policy:**

- Current major: Full support
- Previous major: Security patches for 12 months
- Older: No support

---

### SaaS Upgrades

SaaS customers receive automatic updates:

- Patch releases: Deployed immediately (no action required)
- Minor releases: Deployed with 7-day notice
- Major releases: Opt-in with migration period

**Notification:**

- Email notification before minor/major updates
- In-app banner announcing upcoming changes
- Release notes in documentation

**Opting out (temporarily):**

```bash
# Request upgrade delay (up to 30 days)
curl -X POST https://api.sensei.ai/v1/account/upgrade-delay \
  -d '{"until": "2026-03-01"}'
```

---

### Hybrid Upgrades

#### Control Plane

Updated automatically by Sensei (same as SaaS)

#### Data Plane

Updated manually via Helm:

```bash
# 1. Check current version
helm list -n sensei

# 2. Check available versions
helm search repo sensei/data-plane --versions

# 3. Review release notes
# https://docs.sensei.ai/releases/

# 4. Backup current values
helm get values sensei-data-plane -n sensei > values-backup.yaml

# 5. Perform upgrade
helm upgrade sensei-data-plane sensei/data-plane \
  --version 1.2.0 \
  --namespace sensei \
  -f values.yaml \
  --wait

# 6. Verify upgrade
kubectl get pods -n sensei
sensei health check
```

---

### Self-Hosted Upgrades

#### Pre-Upgrade Checklist

```yaml
checklist:
  - [ ] Review release notes for breaking changes
  - [ ] Backup database
  - [ ] Backup configuration
  - [ ] Test upgrade in staging environment
  - [ ] Schedule maintenance window
  - [ ] Notify users
  - [ ] Ensure no active migrations (or they're paused)
```

#### Upgrade Procedure

```bash
# 1. Pause active migrations
sensei migrations pause --all

# 2. Create database backup
kubectl exec postgresql-0 -n sensei -- \
  pg_dump -U sensei sensei > backup-pre-upgrade.sql

# 3. Update Helm repo
helm repo update

# 4. Dry-run upgrade
helm upgrade sensei sensei/sensei \
  --version 1.2.0 \
  --namespace sensei \
  -f values.yaml \
  --dry-run

# 5. Perform upgrade
helm upgrade sensei sensei/sensei \
  --version 1.2.0 \
  --namespace sensei \
  -f values.yaml \
  --wait \
  --timeout 15m

# 6. Run database migrations (if required)
kubectl exec -it sensei-api-0 -n sensei -- \
  sensei db migrate

# 7. Verify upgrade
sensei health check
kubectl get pods -n sensei

# 8. Resume migrations
sensei migrations resume --all
```

---

### Database Migrations

Some upgrades require database schema changes:

```bash
# Check pending migrations
sensei db migrate --status

# Run migrations
sensei db migrate

# Rollback if needed (last migration only)
sensei db migrate --rollback
```

**Automatic migrations:**

```yaml
# values.yaml - run migrations on startup
controlPlane:
  api:
    autoMigrate: true
```

---

### Rolling Updates

Sensei uses rolling updates by default:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```

**Behavior:**

- Old pods replaced gradually
- No downtime during update
- Migrations continue running
- New pods must pass health checks before old pods terminate

---

### Rollback

If an upgrade causes issues:

#### Helm Rollback

```bash
# View history
helm history sensei -n sensei

# Rollback to previous revision
helm rollback sensei 2 -n sensei

# Rollback to specific revision
helm rollback sensei 5 -n sensei
```

#### Database Rollback

```bash
# Restore from backup
kubectl exec -i postgresql-0 -n sensei -- \
  psql -U sensei sensei < backup-pre-upgrade.sql
```

#### Full Rollback Procedure

```bash
# 1. Pause all migrations
sensei migrations pause --all

# 2. Rollback Helm release
helm rollback sensei -n sensei

# 3. Rollback database (if migrations were run)
sensei db migrate --rollback

# 4. Verify rollback
sensei health check

# 5. Resume migrations
sensei migrations resume --all
```

---

### Air-Gapped Upgrades

```bash
# On connected machine: download new version
sensei bundle create --version 1.2.0 --output sensei-1.2.0.tar.gz

# Transfer to air-gapped environment

# Load new images
tar -xzf sensei-1.2.0.tar.gz
./load-images.sh --registry registry.internal

# Upgrade via Helm (pointing to local registry)
helm upgrade sensei sensei/sensei \
  --version 1.2.0 \
  --set global.imageRegistry=registry.internal
```

---

### Version Compatibility

| Component            | Compatibility        |
| -------------------- | -------------------- |
| Control ↔ Data Plane | Same major, ±1 minor |
| API ↔ SDK            | Same major version   |
| CLI ↔ API            | Same major, ±2 minor |

**Example:** Control plane 1.3.x works with data plane 1.2.x-1.4.x

---

### Notifications

Subscribe to upgrade notifications:

```yaml
# Configure in settings
notifications:
  upgrades:
    email: ops@example.com
    slack: '#sensei-ops'
    advance_notice_days: 14
```

→ [Backup & Recovery](backup-recovery.md)
→ [High Availability](high-availability.md)
→ [Deployment Overview](README.md)
