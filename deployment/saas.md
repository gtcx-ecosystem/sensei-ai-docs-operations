# SaaS Deployment

## Fully Managed Sensei

SaaS is the fastest way to start with Sensei. We manage all infrastructure, updates, and scaling — you just connect your databases and migrate.

---

### How It Works

```text
┌─────────────────────────────────────────────────────────────────────┐
│                         YOUR ENVIRONMENT                             │
│  ┌─────────────────┐                    ┌─────────────────┐         │
│  │  Source DB      │                    │  Target DB      │         │
│  │  (your VPC)     │                    │  (your VPC)     │         │
│  └────────┬────────┘                    └────────┬────────┘         │
│           │                                      │                   │
└───────────┼──────────────────────────────────────┼───────────────────┘
            │ Secure connection                    │
            │ (TLS 1.3)                           │
            ↓                                      ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         SENSEI CLOUD                                 │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     Sensei Platform                          │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │   │
│  │  │  MABA   │  │  KORA   │  │  AMANI  │  │ Agents  │        │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Managed by Sensei: Infrastructure, updates, scaling, security     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Getting Started

#### 1. Create Account

Sign up at [sensei.ai/signup](https://sensei.ai/signup)

#### 2. Connect Source Database

```yaml
# Example: PostgreSQL source
source:
  type: postgresql
  host: your-db.example.com
  port: 5432
  database: production
  username: sensei_reader
  ssl_mode: require
```

**Network requirements:**

- Database must be accessible from internet (or use VPC peering)
- Firewall must allow connections from Sensei IPs
- TLS required for all connections

#### 3. Connect Target Database

```yaml
# Example: Snowflake target
target:
  type: snowflake
  account: xy12345.us-east-1
  warehouse: MIGRATION_WH
  database: MIGRATED_DATA
```

#### 4. Start Migrating

- Create migration in dashboard
- Review and approve plan
- Start execution
- Monitor via dashboard or AMANI

---

### Sensei IP Addresses

Allowlist these IPs in your firewall:

| Region       | IP Ranges                       |
| ------------ | ------------------------------- |
| US East      | `52.20.0.0/16`, `54.80.0.0/16`  |
| US West      | `52.40.0.0/16`, `54.200.0.0/16` |
| EU West      | `52.30.0.0/16`, `54.170.0.0/16` |
| AP Southeast | `52.60.0.0/16`, `54.250.0.0/16` |

Or use VPC peering for private connectivity (Professional/Enterprise plans).

---

### Data Handling

| Data Type                | Handling                                |
| ------------------------ | --------------------------------------- |
| **Migration data**       | Streamed through Sensei, not stored     |
| **Schema metadata**      | Stored (encrypted) for planning         |
| **Statistical profiles** | Stored (encrypted) for pattern matching |
| **Credentials**          | Stored in encrypted vault, never logged |
| **Logs**                 | Retained 30 days, then deleted          |

**Data residency:**

- US region: Data processed in AWS us-east-1
- EU region: Data processed in AWS eu-west-1
- Other regions available on Enterprise plans

---

### Security

| Control               | Implementation                          |
| --------------------- | --------------------------------------- |
| Encryption in transit | TLS 1.3 required                        |
| Encryption at rest    | AES-256                                 |
| Authentication        | API keys, OAuth 2.0, SSO (Enterprise)   |
| Access control        | Role-based (RBAC)                       |
| Audit logging         | All actions logged, 1-year retention    |
| Compliance            | SOC 2 Type II, GDPR, HIPAA (Enterprise) |

---

### Limits

| Resource              | Starter | Professional | Enterprise |
| --------------------- | ------- | ------------ | ---------- |
| Concurrent migrations | 2       | 10           | Unlimited  |
| Data volume/month     | 100 GB  | 1 TB         | Unlimited  |
| API requests/day      | 10,000  | 100,000      | Unlimited  |
| Retention             | 30 days | 90 days      | Custom     |
| Support               | Email   | Email + Chat | Dedicated  |

---

### When to Choose SaaS

**Choose SaaS if:**

- You want the fastest time to value
- Your databases are internet-accessible (or can use VPC peering)
- You don't have strict data residency requirements beyond US/EU
- You prefer managed infrastructure

**Consider Hybrid or Self-Hosted if:**

- Data cannot leave your network
- Air-gapped environment required
- Specific compliance requirements (FedRAMP, etc.)
- You need complete infrastructure control

→ [Hybrid Deployment](hybrid.md)
→ [Self-Hosted Deployment](self-hosted.md)
→ [Network Configuration](network.md)
