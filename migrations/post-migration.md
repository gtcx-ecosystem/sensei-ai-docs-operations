# Post-Migration

## Cleanup, Handoff, and Decommissioning

After successful migration and validation, complete these post-migration activities to close out the project properly.

---

### Post-Migration Checklist

```yaml
post_migration_checklist:
  immediate: # Within 24 hours
    - [ ] Confirm target is serving production traffic
    - [ ] Verify monitoring shows healthy metrics
    - [ ] Distribute Behavioral Equivalence Certificate
    - [ ] Document any exceptions or known issues

  short_term: # Within 1 week
    - [ ] Complete performance baseline comparison
    - [ ] Gather user feedback
    - [ ] Clean up temporary resources
    - [ ] Archive migration artifacts

  medium_term: # Within 1 month
    - [ ] Decommission source database
    - [ ] Update documentation and runbooks
    - [ ] Conduct retrospective
    - [ ] Close project formally
```

---

### Certificate Distribution

Deliver the Behavioral Equivalence Certificate to stakeholders:

```bash
# Download certificate
curl https://api.sensei.ai/v1/migrations/{id}/certificate \
  -H "Authorization: Bearer sk_live_abc123" \
  -o migration_certificate.pdf
```

**Recipients:**

- Technical lead (for records)
- Compliance officer (for audit file)
- Business sponsor (for project close)
- IT audit (for compliance evidence)

The certificate includes:

- Validation results (all five perspectives)
- Coverage metrics
- Hash chain evidence
- Digital signature
- Any documented exceptions

---

### Performance Baseline

Compare source and target performance:

```bash
curl https://api.sensei.ai/v1/migrations/{id}/performance-comparison
```

```json
{
  "comparison": {
    "query_latency": {
      "source_p95_ms": 145,
      "target_p95_ms": 87,
      "improvement": "40%"
    },
    "throughput": {
      "source_qps": 1200,
      "target_qps": 2400,
      "improvement": "100%"
    },
    "storage": {
      "source_gb": 127,
      "target_gb": 98,
      "reduction": "23%"
    }
  },
  "notes": [
    "Target uses columnar storage, improving analytics queries",
    "Compression enabled on target",
    "Indexes optimized for actual query patterns"
  ]
}
```

Document this comparison for stakeholders.

---

### Resource Cleanup

#### Sensei Resources

```bash
# Archive migration (keeps records, releases resources)
curl -X POST https://api.sensei.ai/v1/migrations/{id}/archive

# Delete checkpoints (if not needed for audit)
curl -X DELETE https://api.sensei.ai/v1/migrations/{id}/checkpoints

# Release temporary storage
curl -X POST https://api.sensei.ai/v1/migrations/{id}/cleanup
```

#### Infrastructure Resources

- Terminate any temporary compute (extra workers)
- Delete staging environments used for testing
- Remove temporary network routes (VPC peering for migration)
- Clean up temporary S3 buckets/storage

---

### Artifact Archival

Archive for compliance and future reference:

| Artifact                           | Retention  | Location           |
| ---------------------------------- | ---------- | ------------------ |
| Behavioral Equivalence Certificate | 7 years    | Compliance archive |
| Migration plan (approved)          | 7 years    | Project archive    |
| Validation results                 | 7 years    | Compliance archive |
| Execution logs                     | 2 years    | Log archive        |
| Mapping documentation              | Indefinite | Knowledge base     |
| Error resolution records           | 2 years    | Project archive    |

```bash
# Export all artifacts
curl https://api.sensei.ai/v1/migrations/{id}/export \
  -H "Authorization: Bearer sk_live_abc123" \
  -o migration_artifacts.zip
```

---

### Source Decommissioning

**Wait for confidence period** — Don't decommission immediately.

Recommended timeline:

- **Week 1-2:** Keep source read-only, available for rollback
- **Week 2-4:** Reduce source to minimal infrastructure
- **Week 4+:** Decommission source completely

**Decommissioning checklist:**

```yaml
source_decommissioning:
  prerequisites:
    - [ ] No applications still connecting to source
    - [ ] Rollback window expired
    - [ ] Final backup taken (for archival)
    - [ ] Stakeholder approval obtained

  steps:
    - [ ] Stop source database
    - [ ] Take final snapshot (archive)
    - [ ] Update DNS/connection strings to error
    - [ ] Wait 1 week for any stragglers
    - [ ] Delete source infrastructure
    - [ ] Release network resources
    - [ ] Close vendor accounts if applicable
```

---

### Documentation Updates

Update organizational documentation:

- **Architecture diagrams** — Show new target database
- **Runbooks** — Update operational procedures
- **Disaster recovery** — Update DR procedures for new target
- **Monitoring** — Update dashboards for new target
- **Oncall documentation** — Update troubleshooting guides

---

### Retrospective

Conduct a retrospective meeting:

**What went well:**

- Autonomous error recovery worked as expected
- Validation caught the date format issue before production
- Stakeholder communication via AMANI was effective

**What could improve:**

- Initial resource estimates were too low
- Network latency to Snowflake wasn't accounted for
- Some team members weren't trained on AMANI

**Action items:**

- Create template for network planning
- Add Snowflake-specific tuning to playbook
- Schedule AMANI training for new team members

---

### Project Closure

Formal closure activities:

1. **Final report** — Summarize outcomes, metrics, lessons learned
2. **Budget reconciliation** — Compare actual vs. estimated costs
3. **Stakeholder sign-off** — Get formal acceptance
4. **Knowledge transfer** — Ensure operations team is trained
5. **Celebration** — Recognize the team's work

**Final report template:**

```markdown
# Migration Project Final Report

## Summary

- Source: Oracle 19c (HR_PROD)
- Target: PostgreSQL 16 (hr_production)
- Duration: 3 weeks (planning through certification)
- Records migrated: 52,000,000
- Final status: Certified

## Outcomes

- Migration completed on schedule
- Zero production incidents during cutover
- 40% query latency improvement achieved
- 23% storage reduction achieved

## Cost

- Estimated: $15,000
- Actual: $14,230
- Variance: -5% (under budget)

## Lessons Learned

[Include retrospective findings]

## Acknowledgments

[Team members who contributed]
```

→ [Runbooks](runbooks.md)
→ [Operations Overview](README.md)
