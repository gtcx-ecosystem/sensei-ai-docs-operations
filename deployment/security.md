# Security

## Encryption, Authentication, and Compliance

Sensei implements defense-in-depth security across all deployment models. This page covers security controls and how to configure them.

---

### Security Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                       SECURITY LAYERS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  PERIMETER          │ WAF, DDoS protection, IP allowlisting  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  NETWORK            │ TLS 1.3, mTLS, network segmentation    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  AUTHENTICATION     │ API keys, OAuth 2.0, SSO, MFA          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  AUTHORIZATION      │ RBAC, resource-level permissions       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  DATA               │ Encryption at rest (AES-256), in-mem   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  AUDIT              │ Comprehensive logging, tamper-proof    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Encryption

#### In Transit

| Connection          | Encryption    | Configuration           |
| ------------------- | ------------- | ----------------------- |
| Client → API        | TLS 1.3       | Required, no fallback   |
| Internal services   | mTLS          | Certificate-based       |
| Sensei → Databases  | TLS           | Required by default     |
| Agent communication | TLS + signing | Message-level integrity |

```yaml
# Enforce TLS for database connections
source:
  type: postgresql
  ssl_mode: verify-full # Options: disable, require, verify-ca, verify-full
  ssl_ca: /certs/ca.crt
  ssl_cert: /certs/client.crt
  ssl_key: /certs/client.key
```

#### At Rest

| Data                  | Encryption  | Key Management    |
| --------------------- | ----------- | ----------------- |
| Metadata (PostgreSQL) | AES-256     | Platform KMS      |
| Credentials           | AES-256-GCM | Dedicated vault   |
| Checkpoints           | AES-256     | Per-customer keys |
| Logs                  | AES-256     | Platform KMS      |

**Bring Your Own Key (BYOK):**

```yaml
encryption:
  provider: aws-kms # or gcp-kms, azure-keyvault, hashicorp-vault
  keyId: arn:aws:kms:us-east-1:123456789:key/abc-123
```

---

### Authentication

#### API Keys

```yaml
# Create restricted API key
apiKey:
  name: 'migration-automation'
  permissions:
    - migrations:read
    - migrations:write
    - connections:read
  ipAllowlist:
    - 10.0.0.0/8
  expiresAt: 2027-01-01T00:00:00Z
```

#### OAuth 2.0 / OIDC

```yaml
authentication:
  oidc:
    enabled: true
    issuer: https://login.example.com
    clientId: sensei-app
    clientSecret: ${OIDC_CLIENT_SECRET}
    scopes: [openid, profile, email]
```

#### SSO (Enterprise)

Supported providers:

- Okta
- Azure AD
- Google Workspace
- OneLogin
- PingIdentity
- SAML 2.0 (generic)

```yaml
authentication:
  sso:
    enabled: true
    provider: okta
    domain: example.okta.com
    clientId: ${OKTA_CLIENT_ID}
    clientSecret: ${OKTA_CLIENT_SECRET}
```

#### MFA

```yaml
authentication:
  mfa:
    required: true
    methods: [totp, webauthn]
    gracePerioddDays: 7
```

---

### Authorization (RBAC)

#### Built-in Roles

| Role        | Permissions                             |
| ----------- | --------------------------------------- |
| `viewer`    | Read migrations, connections, logs      |
| `operator`  | + Start/pause/resume migrations         |
| `developer` | + Create/modify migrations, connections |
| `admin`     | + Manage users, billing, settings       |
| `owner`     | Full access including deletion          |

#### Custom Roles

```yaml
roles:
  - name: migration-approver
    permissions:
      - migrations:read
      - migrations:approve
      - mappings:read
      - mappings:write
```

#### Resource-Level Permissions

```yaml
# Restrict user to specific migrations
userPermissions:
  user: alice@example.com
  resources:
    - type: migration
      ids: [mig_abc, mig_def]
      permissions: [read, write, execute]
```

---

### Secrets Management

#### Supported Secret Stores

| Store               | Configuration                            |
| ------------------- | ---------------------------------------- |
| HashiCorp Vault     | `vault://secret/sensei/db-credentials`   |
| AWS Secrets Manager | `aws-sm://sensei/db-credentials`         |
| GCP Secret Manager  | `gcp-sm://projects/123/secrets/db-creds` |
| Azure Key Vault     | `azure-kv://sensei-vault/db-credentials` |
| Kubernetes Secrets  | `k8s://sensei/db-credentials`            |

```yaml
# Reference secrets in configuration
source:
  type: postgresql
  username: ${vault://secret/sensei/source-db#username}
  password: ${vault://secret/sensei/source-db#password}
```

---

### Audit Logging

All security-relevant events are logged:

```json
{
  "timestamp": "2026-02-02T14:23:00Z",
  "event": "authentication.success",
  "actor": {
    "type": "user",
    "id": "user_abc123",
    "email": "alice@example.com",
    "ip": "10.0.1.50"
  },
  "resource": {
    "type": "migration",
    "id": "mig_xyz789"
  },
  "action": "start",
  "result": "success",
  "metadata": {
    "userAgent": "Sensei-SDK/1.0",
    "mfaUsed": true
  }
}
```

**Audit log retention:** 1 year (configurable up to 7 years)

**Export to SIEM:**

```yaml
audit:
  export:
    - type: splunk
      endpoint: https://splunk.example.com:8088
      token: ${SPLUNK_HEC_TOKEN}
    - type: s3
      bucket: audit-logs
      prefix: sensei/
```

---

### Compliance

| Standard      | Status      | Notes                           |
| ------------- | ----------- | ------------------------------- |
| SOC 2 Type II | Certified   | Annual audit                    |
| GDPR          | Compliant   | EU data residency available     |
| HIPAA         | Available   | Enterprise, BAA required        |
| PCI DSS       | Partial     | Sensei does not store card data |
| FedRAMP       | In progress | Expected 2026                   |
| ISO 27001     | Certified   | Annual audit                    |

**Compliance documentation:**

- SOC 2 report available on request (NDA required)
- GDPR DPA template available
- HIPAA BAA template available (Enterprise)

---

### Security Best Practices

1. **Rotate credentials regularly** — API keys quarterly, service accounts monthly
2. **Use least privilege** — Grant minimum required permissions
3. **Enable MFA** — Require for all human users
4. **Monitor audit logs** — Set up alerts for suspicious activity
5. **Keep software updated** — Apply security patches promptly
6. **Use private networking** — VPC peering over public internet
7. **Encrypt everything** — Enable encryption at rest and in transit

→ [Network Configuration](network.md)
→ [Authentication](../../developers/integration/authentication.md)
→ [Deployment Overview](README.md)
