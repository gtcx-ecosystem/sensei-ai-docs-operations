# Security Checklist

## Security Review Checklist

Use this checklist to ensure your Sensei deployment meets security requirements.

---

### Authentication & Access Control

#### API Keys

- [ ] API keys stored securely (not in code)
- [ ] API keys have minimal required permissions
- [ ] API key rotation policy defined
- [ ] Unused API keys revoked
- [ ] API key usage monitored

#### User Access

- [ ] SSO/SAML configured (Professional+)
- [ ] MFA enabled for all users
- [ ] User access regularly reviewed
- [ ] Principle of least privilege applied
- [ ] Offboarded users removed promptly

#### Role-Based Access

- [ ] Roles defined and documented
- [ ] Users assigned appropriate roles
- [ ] Admin access limited
- [ ] Service accounts audited

---

### Network Security

#### Connectivity

- [ ] Connections use TLS 1.2+
- [ ] SSL certificates valid and trusted
- [ ] Network segmentation in place
- [ ] Firewall rules configured
- [ ] Unnecessary ports closed

#### IP Restrictions

- [ ] IP allowlist configured (if required)
- [ ] VPN/PrivateLink used for sensitive data
- [ ] Public access disabled (if not needed)
- [ ] Network traffic encrypted

#### Database Connections

- [ ] Database credentials encrypted
- [ ] Credentials not stored in plaintext
- [ ] Connection strings secure
- [ ] Read-only access for source (when possible)

---

### Data Protection

#### Encryption at Rest

- [ ] Data encrypted at rest (AES-256)
- [ ] Encryption keys managed securely
- [ ] Key rotation policy in place
- [ ] Customer-managed keys (if required)

#### Encryption in Transit

- [ ] All API traffic over HTTPS
- [ ] Database connections encrypted
- [ ] Internal communication encrypted
- [ ] Certificate management automated

#### Data Handling

- [ ] PII identified and documented
- [ ] Sensitive data masked in logs
- [ ] Data retention policies configured
- [ ] Data deletion procedures defined

---

### Audit & Compliance

#### Logging

- [ ] Audit logging enabled
- [ ] Logs include required events
- [ ] Log retention meets requirements
- [ ] Logs protected from tampering
- [ ] SIEM integration configured (if required)

#### Monitoring

- [ ] Security monitoring active
- [ ] Alerts configured for anomalies
- [ ] Failed login attempts monitored
- [ ] API usage monitored
- [ ] Incident response plan defined

#### Documentation

- [ ] Security policies documented
- [ ] Data flow diagrams created
- [ ] Access control matrix maintained
- [ ] Incident response runbook ready

---

### Deployment Security

#### SaaS Deployment

- [ ] Organization security settings reviewed
- [ ] IP restrictions configured (if needed)
- [ ] SSO configured (Professional+)
- [ ] Webhook secrets configured

#### Hybrid Deployment

- [ ] Data plane in secure VPC
- [ ] No public internet exposure
- [ ] Secrets stored in vault
- [ ] Container images scanned
- [ ] Kubernetes RBAC configured

#### Self-Hosted Deployment

- [ ] Infrastructure hardened
- [ ] OS security patches current
- [ ] Container runtime secured
- [ ] Network policies enforced
- [ ] Secrets management configured
- [ ] Backup encryption enabled

---

### Operational Security

#### Secrets Management

- [ ] Secrets stored in vault/secrets manager
- [ ] No hardcoded credentials
- [ ] Secrets rotated regularly
- [ ] Access to secrets audited
- [ ] Emergency credential procedures defined

#### Vulnerability Management

- [ ] Regular security scans performed
- [ ] Dependencies kept up to date
- [ ] CVE monitoring active
- [ ] Patch management process defined
- [ ] Penetration testing scheduled

#### Incident Response

- [ ] Incident response plan documented
- [ ] Contact list current
- [ ] Communication plan ready
- [ ] Recovery procedures tested
- [ ] Post-incident review process defined

---

### Compliance-Specific

#### HIPAA (Healthcare)

- [ ] BAA executed with Sensei
- [ ] PHI handling procedures documented
- [ ] Access controls meet requirements
- [ ] Audit trail comprehensive
- [ ] Breach notification plan ready

#### PCI DSS (Payment Data)

- [ ] Cardholder data not processed (typically)
- [ ] If processed, requirements documented
- [ ] Scope minimized

#### GDPR (EU Data)

- [ ] DPA signed with Sensei
- [ ] Data processing purposes documented
- [ ] Data subject rights procedures ready
- [ ] Cross-border transfer mechanisms in place

#### SOC 2

- [ ] Sensei SOC 2 report reviewed
- [ ] Complementary controls implemented
- [ ] Control objectives documented

---

### Vendor Security

#### Sensei Assessment

- [ ] Sensei SOC 2 report reviewed
- [ ] Sensei security questionnaire completed
- [ ] Sensei penetration test results reviewed
- [ ] Contract terms reviewed
- [ ] SLA acceptable

#### Third-Party Risk

- [ ] Subprocessors identified
- [ ] Data flow to third parties documented
- [ ] Third-party security assessed

---

### Review Schedule

| Review                 | Frequency     |
| ---------------------- | ------------- |
| Access review          | Quarterly     |
| API key rotation       | Every 90 days |
| Security configuration | Monthly       |
| Vulnerability scan     | Weekly        |
| Penetration test       | Annually      |
| Policy review          | Annually      |

---

### Sign-Off

| Reviewer | Role               | Date | Signature |
| -------- | ------------------ | ---- | --------- |
|          | Security Lead      |      |           |
|          | Data Owner         |      |           |
|          | Compliance Officer |      |           |
|          | IT Manager         |      |           |

→ [Security Overview](../../enterprise/business/compliance/security-overview.md) → [Compliance](../../enterprise/business/compliance/) → [Compliance Mapping](compliance-mapping.md)
