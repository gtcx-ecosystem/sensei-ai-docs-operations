# Migration Checklist

## Complete Migration Checklist

Use this checklist to ensure a successful migration.

---

### Pre-Migration Planning

#### Discovery

- [ ] Identify all source systems to migrate
- [ ] Document current data volumes
- [ ] Inventory tables, views, stored procedures
- [ ] Identify data owners and stakeholders
- [ ] Document existing integrations and dependencies

#### Requirements

- [ ] Define migration success criteria
- [ ] Establish acceptable downtime window
- [ ] Determine data validation requirements
- [ ] Identify compliance/regulatory requirements
- [ ] Set performance expectations

#### Timeline

- [ ] Establish target completion date
- [ ] Identify key milestones
- [ ] Plan for testing phases
- [ ] Schedule cutover window
- [ ] Allocate rollback time buffer

---

### Technical Preparation

#### Source System

- [ ] Create read-only migration user
- [ ] Grant required permissions
- [ ] Test connectivity from Sensei
- [ ] Document source schema
- [ ] Identify source system maintenance windows
- [ ] Enable CDC if using incremental migration

#### Target System

- [ ] Provision target database
- [ ] Create migration user with write permissions
- [ ] Configure appropriate storage
- [ ] Set up monitoring
- [ ] Test connectivity from Sensei
- [ ] Disable unnecessary constraints during load (optional)

#### Network

- [ ] Verify network connectivity
- [ ] Configure firewall rules
- [ ] Set up VPN/PrivateLink if required
- [ ] Test bandwidth and latency
- [ ] Plan for network maintenance windows

---

### Sensei Configuration

#### Connections

- [ ] Create source connection
- [ ] Test source connection
- [ ] Create target connection
- [ ] Test target connection
- [ ] Configure SSL/TLS if required

#### Migration Setup

- [ ] Run MABA schema analysis
- [ ] Review table selection
- [ ] Verify column mappings
- [ ] Configure transformations
- [ ] Set batch sizes and worker counts
- [ ] Enable checkpointing
- [ ] Configure error handling

#### Validation

- [ ] Enable required validation types
- [ ] Configure validation thresholds
- [ ] Set up Time-Travel Testing (if using)
- [ ] Define validation exceptions

#### Notifications

- [ ] Configure webhook endpoints
- [ ] Set up email notifications
- [ ] Test notification delivery
- [ ] Define escalation contacts

---

### Testing

#### Dry Run

- [ ] Run migration on subset of data
- [ ] Verify transformations work correctly
- [ ] Check validation results
- [ ] Measure performance
- [ ] Test error recovery

#### Full Test

- [ ] Run full migration to test environment
- [ ] Validate all data
- [ ] Test application connectivity
- [ ] Verify integrations work
- [ ] Document any issues and resolutions

#### Performance Testing

- [ ] Measure throughput
- [ ] Identify bottlenecks
- [ ] Tune configuration if needed
- [ ] Verify meets requirements

---

### Pre-Cutover

#### Stakeholder Communication

- [ ] Notify all stakeholders of cutover schedule
- [ ] Confirm team availability
- [ ] Distribute contact information
- [ ] Send final reminder 24 hours before

#### Final Preparation

- [ ] Verify source data is current
- [ ] Confirm target system is ready
- [ ] Test rollback procedure
- [ ] Prepare monitoring dashboards
- [ ] Have runbook ready

#### Go/No-Go Decision

- [ ] All test migrations successful
- [ ] Validation requirements met
- [ ] Rollback tested and ready
- [ ] Team available and ready
- [ ] Stakeholder approval received

---

### Cutover Execution

#### Pre-Cutover

- [ ] Announce maintenance window
- [ ] Disable application writes to source (if required)
- [ ] Take final source backup
- [ ] Verify CDC is caught up (if using)

#### Migration

- [ ] Start final migration/sync
- [ ] Monitor progress
- [ ] Verify no errors
- [ ] Wait for completion

#### Validation

- [ ] Run KORA validation
- [ ] Verify row counts
- [ ] Check referential integrity
- [ ] Run behavioral validation (if using)
- [ ] Review validation report

#### Switch

- [ ] Update application connection strings
- [ ] Redirect traffic to target
- [ ] Verify applications work
- [ ] Monitor for errors

---

### Post-Cutover

#### Immediate (First Hour)

- [ ] Monitor application error rates
- [ ] Watch database performance
- [ ] Check integration health
- [ ] Verify critical workflows
- [ ] Be ready to rollback

#### Short-Term (First 24 Hours)

- [ ] Keep source available for rollback
- [ ] Monitor for reported issues
- [ ] Verify batch jobs run correctly
- [ ] Check scheduled reports
- [ ] Address any issues

#### Medium-Term (First Week)

- [ ] Continue monitoring
- [ ] Address edge cases
- [ ] Collect user feedback
- [ ] Document any issues
- [ ] Begin source decommission planning

---

### Completion

#### Documentation

- [ ] Document final configuration
- [ ] Archive migration logs
- [ ] Save validation certificates
- [ ] Write lessons learned
- [ ] Update system documentation

#### Cleanup

- [ ] Remove temporary connections
- [ ] Revoke migration credentials
- [ ] Delete temporary data
- [ ] Archive source (per retention policy)
- [ ] Close migration project

#### Sign-Off

- [ ] Obtain stakeholder sign-off
- [ ] Close tickets/issues
- [ ] Celebrate success

---

### Quick Reference

**Critical success factors:**

1. Thorough testing before cutover
2. Clear communication with stakeholders
3. Ready and tested rollback plan
4. Active monitoring during cutover
5. Patience during stabilization

**Common pitfalls to avoid:**

- Rushing the testing phase
- Skipping validation
- No rollback plan
- Decommissioning source too early
- Inadequate stakeholder communication

→ [Best Practices](../../enterprise/resources/best-practices/) → [Security Checklist](security-checklist.md)
