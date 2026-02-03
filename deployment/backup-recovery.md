# Backup & Recovery

## Data Protection and Restore Procedures

Protecting Sensei's data ensures you can recover from failures, accidental deletions, or disasters. This page covers what to back up and how to restore.

---

### What to Back Up

| Data                | Importance | Frequency | Retention |
| ------------------- | ---------- | --------- | --------- |
| **PostgreSQL**      | Critical   | Hourly    | 30 days   |
| **Redis**           | Medium     | Daily     | 7 days    |
| **Object Storage**  | High       | Daily     | 90 days   |
| **Vector Database** | Medium     | Daily     | 30 days   |
| **Configuration**   | Critical   | On change | Forever   |
| **Secrets**         | Critical   | On change | Forever   |

---

### SaaS Backups

Sensei manages backups automatically:

| Data              | RPO    | Retention | Restore Time |
| ----------------- | ------ | --------- | ------------ |
| All customer data | 1 hour | 30 days   | <4 hours     |
| Certificates      | N/A    | 7 years   | Immediate    |

**Request restore:**

```bash
curl -X POST https://api.sensei.ai/v1/account/restore-request \
  -d '{
    "type": "migration",
    "migration_id": "mig_abc123",
    "restore_to": "2026-02-01T00:00:00Z"
  }'
```

---

### Hybrid/Self-Hosted Backups

#### PostgreSQL Backup

**Continuous archiving (recommended):**

```yaml
postgresql:
  backup:
    enabled: true
    schedule: '0 * * * *' # Hourly
    retention: 30

    # WAL archiving for PITR
    walArchiving:
      enabled: true
      destination: s3://backups/wal/

    # Full backup destination
    destination:
      type: s3
      bucket: sensei-backups
      prefix: postgresql/
```

**Manual backup:**

```bash
# Full backup
kubectl exec postgresql-0 -n sensei -- \
  pg_dump -U sensei -Fc sensei > sensei-$(date +%Y%m%d).dump

# Upload to S3
aws s3 cp sensei-$(date +%Y%m%d).dump s3://backups/postgresql/
```

#### Redis Backup

```yaml
redis:
  backup:
    enabled: true
    schedule: '0 0 * * *' # Daily
    retention: 7
    destination:
      type: s3
      bucket: sensei-backups
      prefix: redis/
```

**Manual backup:**

```bash
# Trigger RDB snapshot
kubectl exec redis-0 -n sensei -- redis-cli BGSAVE

# Copy RDB file
kubectl cp sensei/redis-0:/data/dump.rdb ./redis-backup.rdb
```

#### Object Storage Backup

Enable cross-region replication:

```yaml
# MinIO replication
minio:
  replication:
    enabled: true
    target:
      endpoint: s3.us-west-2.amazonaws.com
      bucket: sensei-backups-dr
```

Or sync to backup location:

```bash
# Sync to backup bucket
aws s3 sync s3://sensei-data s3://sensei-backups/objects/
```

#### Vector Database Backup

```bash
# Weaviate backup
curl -X POST http://weaviate:8080/v1/backups/s3 \
  -H "Content-Type: application/json" \
  -d '{
    "id": "backup-20260202",
    "include": ["MigrationPatterns"]
  }'
```

#### Configuration Backup

```bash
# Export Helm values
helm get values sensei -n sensei > values-backup.yaml

# Export secrets (encrypted)
kubectl get secrets -n sensei -o yaml | \
  kubeseal --format yaml > secrets-backup.yaml

# Store in version control or secure storage
```

---

### Backup Automation

**CronJob for database backup:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgresql-backup
  namespace: sensei
spec:
  schedule: '0 */6 * * *' # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16
              command:
                - /bin/sh
                - -c
                - |
                  pg_dump -h postgresql -U sensei -Fc sensei | \
                  aws s3 cp - s3://sensei-backups/postgresql/backup-$(date +%Y%m%d-%H%M).dump
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgresql
                      key: password
          restartPolicy: OnFailure
```

---

### Restore Procedures

#### PostgreSQL Restore

**From dump file:**

```bash
# 1. Stop Sensei services
kubectl scale deployment --all --replicas=0 -n sensei

# 2. Restore database
kubectl exec -i postgresql-0 -n sensei -- \
  pg_restore -U sensei -d sensei --clean < backup.dump

# 3. Restart services
kubectl scale deployment --all --replicas=1 -n sensei
```

**Point-in-time recovery:**

```bash
# 1. Stop services
kubectl scale deployment --all --replicas=0 -n sensei

# 2. Restore to specific time
kubectl exec -it postgresql-0 -n sensei -- \
  pg_restore_pitr --target-time "2026-02-02 10:00:00"

# 3. Restart services
kubectl scale deployment --all --replicas=1 -n sensei
```

#### Redis Restore

```bash
# 1. Stop Redis
kubectl scale statefulset redis --replicas=0 -n sensei

# 2. Copy backup
kubectl cp redis-backup.rdb sensei/redis-0:/data/dump.rdb

# 3. Start Redis
kubectl scale statefulset redis --replicas=1 -n sensei
```

#### Full System Restore

```bash
# 1. Restore infrastructure (if needed)
terraform apply

# 2. Deploy Sensei
helm install sensei sensei/sensei -f values-backup.yaml

# 3. Restore secrets
kubectl apply -f secrets-backup.yaml

# 4. Restore PostgreSQL
# (follow PostgreSQL restore procedure)

# 5. Restore Redis (optional - can rebuild from PostgreSQL)
# (follow Redis restore procedure)

# 6. Restore vector database
curl -X POST http://weaviate:8080/v1/backups/s3/backup-20260202/restore

# 7. Verify
sensei health check
```

---

### Backup Verification

Regularly test your backups:

```bash
# Monthly backup verification
sensei backup verify --type postgresql --backup-id backup-20260202

# Or restore to test environment
helm install sensei-test sensei/sensei -f test-values.yaml
kubectl exec -i postgresql-test-0 -- pg_restore < backup.dump
sensei health check --context test
```

---

### Retention Policy

| Environment | Database | Objects | Logs    |
| ----------- | -------- | ------- | ------- |
| Development | 7 days   | 7 days  | 3 days  |
| Staging     | 14 days  | 14 days | 7 days  |
| Production  | 30 days  | 90 days | 30 days |
| Compliance  | 7 years  | 7 years | 7 years |

Configure retention:

```yaml
backup:
  retention:
    postgresql:
      daily: 7
      weekly: 4
      monthly: 12
    objects:
      days: 90
```

→ [High Availability](high-availability.md)
→ [Upgrades](upgrades.md)
→ [Deployment Overview](README.md)
