# Compliance Mapping

## Regulatory Requirement Mapping

How Sensei features map to common compliance frameworks.

---

### SOC 2 Trust Service Criteria

| Criteria  | Requirement                | Sensei Feature                           |
| --------- | -------------------------- | ---------------------------------------- |
| **CC6.1** | Logical access controls    | RBAC, API keys, SSO                      |
| **CC6.2** | Prior to access            | User provisioning, approval workflows    |
| **CC6.3** | New access registration    | User management, audit logging           |
| **CC6.6** | Restrict access            | IP allowlisting, network controls        |
| **CC6.7** | Access removal             | User deprovisioning, key revocation      |
| **CC7.1** | Detect unauthorized access | Audit logs, monitoring, alerts           |
| **CC7.2** | Monitoring activities      | Real-time monitoring, dashboards         |
| **CC7.3** | Evaluate anomalies         | Anomaly detection, alerting              |
| **CC7.4** | Incident response          | Incident notification, support           |
| **CC8.1** | Change management          | Version control, release process         |
| **A1.1**  | System availability        | 99.9%+ SLA, redundancy                   |
| **A1.2**  | Backup and recovery        | Automated backups, disaster recovery     |
| **C1.1**  | Confidentiality            | Encryption, access controls              |
| **C1.2**  | Disposal                   | Data retention policies, secure deletion |
| **PI1.1** | Data integrity             | Validation, checksums, certificates      |

---

### GDPR Articles

| Article        | Requirement               | Sensei Feature                         |
| -------------- | ------------------------- | -------------------------------------- |
| **Art. 5**     | Processing principles     | Purpose limitation, data minimization  |
| **Art. 6**     | Lawful processing         | DPA, documented basis                  |
| **Art. 12**    | Transparent information   | Privacy policy, documentation          |
| **Art. 17**    | Right to erasure          | Data deletion capabilities             |
| **Art. 20**    | Data portability          | Export functionality                   |
| **Art. 25**    | Data protection by design | Encryption, privacy-preserving         |
| **Art. 28**    | Processor requirements    | DPA available, subprocessor list       |
| **Art. 30**    | Records of processing     | Audit logs, processing records         |
| **Art. 32**    | Security of processing    | Encryption, access control, monitoring |
| **Art. 33**    | Breach notification       | Incident response, <72hr notification  |
| **Art. 35**    | Impact assessment         | Documentation for DPIA                 |
| **Art. 44-49** | International transfers   | SCCs, adequacy decisions               |

---

### HIPAA Security Rule

| Standard                 | Requirement           | Sensei Feature                             |
| ------------------------ | --------------------- | ------------------------------------------ |
| **164.312(a)(1)**        | Access control        | RBAC, user authentication                  |
| **164.312(a)(2)(i)**     | Unique user ID        | Individual accounts, no shared credentials |
| **164.312(a)(2)(ii)**    | Emergency access      | Break-glass procedures                     |
| **164.312(a)(2)(iii)**   | Automatic logoff      | Session timeout                            |
| **164.312(a)(2)(iv)**    | Encryption            | AES-256 at rest                            |
| **164.312(b)**           | Audit controls        | Comprehensive audit logging                |
| **164.312(c)(1)**        | Integrity             | Data validation, checksums                 |
| **164.312(c)(2)**        | Authentication        | Message authentication                     |
| **164.312(d)**           | Person authentication | MFA, SSO                                   |
| **164.312(e)(1)**        | Transmission security | TLS 1.3 encryption                         |
| **164.312(e)(2)(i)**     | Integrity controls    | Checksums, validation                      |
| **164.312(e)(2)(ii)**    | Encryption            | TLS encryption                             |
| **164.308(a)(1)(ii)(D)** | Activity review       | Audit log review                           |
| **164.310(d)(1)**        | Device controls       | Secure infrastructure                      |

---

### PCI DSS v4.0

| Requirement | Description                        | Sensei Feature                 |
| ----------- | ---------------------------------- | ------------------------------ |
| **3.4**     | Render PAN unreadable              | Data not stored (pass-through) |
| **4.2**     | Protect cardholder data in transit | TLS 1.2+ encryption            |
| **7.1**     | Limit access                       | RBAC, least privilege          |
| **8.2**     | Identify users                     | Unique IDs, authentication     |
| **8.3**     | Strong authentication              | MFA support                    |
| **10.1**    | Audit trails                       | Comprehensive logging          |
| **10.2**    | Automated audit trails             | Automatic log generation       |
| **10.3**    | Audit trail entries                | Required fields captured       |
| **10.5**    | Secure audit trails                | Log integrity protection       |
| **12.10**   | Incident response                  | Incident response plan         |

**Note:** Sensei typically does not process cardholder data directly. If your migration includes payment data, additional controls may be required.

---

### ISO 27001 Controls

| Control    | Description              | Sensei Feature                |
| ---------- | ------------------------ | ----------------------------- |
| **A.5.1**  | Policies                 | Security documentation        |
| **A.6.1**  | Internal organization    | Clear responsibilities        |
| **A.7.1**  | Prior to employment      | Background checks (Anthropic) |
| **A.8.1**  | Asset management         | Asset inventory               |
| **A.9.1**  | Access control policy    | RBAC, documented policies     |
| **A.9.2**  | User access management   | Provisioning, deprovisioning  |
| **A.9.4**  | System access control    | Authentication, authorization |
| **A.10.1** | Cryptographic controls   | Encryption policies           |
| **A.11.1** | Physical security        | Cloud provider controls       |
| **A.12.1** | Operational procedures   | Documented procedures         |
| **A.12.4** | Logging and monitoring   | Audit logs, monitoring        |
| **A.12.6** | Vulnerability management | Patching, scanning            |
| **A.13.1** | Network security         | Network controls, TLS         |
| **A.14.2** | Security in development  | SDLC security                 |
| **A.16.1** | Incident management      | Incident response             |
| **A.18.1** | Compliance               | Regular audits                |

---

### FedRAMP (In Progress)

| Control Family                       | Status      | Notes                  |
| ------------------------------------ | ----------- | ---------------------- |
| Access Control (AC)                  | In Progress | RBAC, MFA implemented  |
| Audit (AU)                           | In Progress | Logging implemented    |
| Configuration Management (CM)        | In Progress |                        |
| Contingency Planning (CP)            | In Progress | DR implemented         |
| Identification & Authentication (IA) | In Progress |                        |
| Incident Response (IR)               | In Progress |                        |
| System Protection (SC)               | In Progress | Encryption implemented |

**Expected authorization:** FedRAMP Moderate (2026)

---

### Industry-Specific

#### Financial Services (FFIEC)

| Requirement         | Sensei Feature        |
| ------------------- | --------------------- |
| Authentication      | MFA, SSO              |
| Access control      | RBAC                  |
| Encryption          | AES-256, TLS 1.3      |
| Audit trail         | Comprehensive logging |
| Business continuity | DR, backups           |

#### Healthcare (HITRUST)

| Domain                 | Sensei Feature             |
| ---------------------- | -------------------------- |
| Information Protection | Encryption, access control |
| Endpoint Protection    | N/A (SaaS)                 |
| Network Protection     | TLS, network controls      |
| Access Control         | RBAC, authentication       |
| Audit Logging          | Comprehensive logs         |

---

### Compliance Documentation

Available upon request (NDA may be required):

| Document                 | Purpose                  |
| ------------------------ | ------------------------ |
| SOC 2 Type II Report     | Independent audit        |
| ISO 27001 Certificate    | Certification            |
| Penetration Test Summary | Security testing         |
| Security Whitepaper      | Architecture overview    |
| DPA                      | Data processing terms    |
| BAA                      | HIPAA business associate |
| Subprocessor List        | Third-party services     |
| SIG Questionnaire        | Security assessment      |

**Request:** compliance@sensei.ai

---

### Compliance by Plan

| Compliance | Starter | Professional | Enterprise     |
| ---------- | ------- | ------------ | -------------- |
| SOC 2      | Yes     | Yes          | Yes            |
| ISO 27001  | Yes     | Yes          | Yes            |
| GDPR       | Yes     | Yes          | Yes            |
| HIPAA      | No      | No           | Yes (with BAA) |
| PCI DSS    | Partial | Partial      | Partial        |
| FedRAMP    | No      | No           | In Progress    |

→ [Compliance](../../enterprise/business/compliance/README.md)
→ [Security Overview](../../enterprise/business/compliance/security-overview.md)
→ [Security Checklist](security-checklist.md)
