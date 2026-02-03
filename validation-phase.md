# Validation Phase

## Post-Migration Verification

After execution completes, Sensei runs comprehensive validation to verify the migration succeeded. This phase produces the evidence needed for regulatory compliance and stakeholder confidence.

---

### Validation Sequence

```
Execution Complete
        │
        ↓
┌───────────────────┐
│    STRUCTURAL     │ ← Schema matches plan?
│    VALIDATION     │   Tables, columns, constraints
└─────────┬─────────┘
          │ Pass
          ↓
┌───────────────────┐
│   STATISTICAL     │ ← Data distributions match?
│   VALIDATION      │   Counts, ranges, patterns
└─────────┬─────────┘
          │ Pass
          ↓
┌───────────────────┐
│   REFERENTIAL     │ ← Relationships intact?
│   VALIDATION      │   FK resolution, orphans
└─────────┬─────────┘
          │ Pass
          ↓
┌───────────────────┐
│    SEMANTIC       │ ← Business rules pass?
│    VALIDATION     │   Domain constraints, plausibility
└─────────┬─────────┘
          │ Pass
          ↓
┌───────────────────┐
│   BEHAVIORAL      │ ← Same behavior as source?
│   VALIDATION      │   Time-Travel Testing
└─────────┬─────────┘
          │ Pass
          ↓
    CERTIFICATION
```

---

### What Each Validation Checks

| Validation | Checks | Pass Criteria |
|------------|--------|---------------|
| **Structural** | Tables exist, columns match, constraints active | 100% match to plan |
| **Statistical** | Row counts, null rates, value distributions | Within configured tolerance |
| **Referential** | FK resolution, cascade behavior | 100% FK integrity |
| **Semantic** | Business rules, domain constraints | 94-98%+ pass rate |
| **Behavioral** | Historical query replay | Business-equivalent results |

→ See [KORA Multi-Source Validation](../components/kora/multi-source-validation.md) for details

---

### Validation Configuration

```yaml
validation:
  # Which validations to run
  enabled:
    structural: true
    statistical: true
    referential: true
    semantic: true
    behavioral: true  # Time-Travel Testing
  
  # Tolerance settings
  tolerances:
    row_count_variance: 0       # Exact match required
    null_rate_variance: 0.001   # 0.1% tolerance
    numeric_epsilon: 0.0001     # Floating point tolerance
    distribution_threshold: 0.95 # K-S test p-value
  
  # Behavioral validation
  behavioral:
    checkpoints_to_replay: 7
    queries_per_checkpoint: 2000
    fidelity_level: semantic    # exact | semantic | business
    coverage_target: 0.8        # 80% table coverage
  
  # Failure handling
  on_failure:
    structural: fail_immediately
    statistical: flag_and_continue
    referential: fail_immediately
    semantic: flag_and_continue
    behavioral: flag_and_continue
```

---

### Validation Results

```yaml
validation_results:
  overall_status: passed
  overall_confidence: 0.9892
  
  structural:
    status: passed
    checks: 247
    passed: 247
    failed: 0
    
  statistical:
    status: passed
    columns_checked: 3891
    within_tolerance: 3884
    flagged: 7
    flagged_details:
      - column: orders.total_amount
        issue: "Mean variance 0.003% (expected <0.01%)"
        severity: low
        
  referential:
    status: passed
    relationships_checked: 236
    integrity_violations: 0
    
  semantic:
    status: passed
    rules_checked: 12847
    passed: 12534
    flagged: 313
    pass_rate: 0.9756
    
  behavioral:
    status: passed
    checkpoints_replayed: 7
    queries_replayed: 11943
    exact_matches: 11891
    semantic_matches: 47
    divergences: 5
    pass_rate: 0.9996
    divergence_details:
      - query_id: q_47291
        issue: "Floating point accumulation difference (0.0002%)"
        classification: acceptable_variance
```

---

### Handling Validation Failures

#### Structural Failures
**Always critical** — indicates infrastructure problem.

Actions:
1. Check target schema was created correctly
2. Verify no manual changes during migration
3. Re-run schema creation if needed
4. Re-migrate affected tables

#### Statistical Failures
**Usually minor** — may indicate transformation issues.

Actions:
1. Review flagged columns
2. Determine if variance is acceptable
3. Mark as accepted (with reason) or investigate
4. Re-migrate affected records if needed

#### Referential Failures
**Critical** — data integrity at risk.

Actions:
1. Identify orphan records
2. Determine cause (missing parent records, FK mapping error)
3. Fix and re-migrate affected records
4. Re-run referential validation

#### Semantic Failures
**Variable severity** — depends on rule importance.

Actions:
1. Review flagged records
2. Classify as true failure vs. false positive
3. Add exceptions for known acceptable cases
4. Re-validate with updated rules

#### Behavioral Failures
**Most important** — indicates functional difference.

Actions:
1. Investigate each divergence
2. Classify as migration defect vs. acceptable variance
3. Fix migration defects and re-migrate
4. Document accepted variances

---

### Timeline

| Dataset Size | Structural | Statistical | Referential | Semantic | Behavioral | Total |
|--------------|-----------|-------------|-------------|----------|------------|-------|
| Small (<1M) | 20s | 5 min | 2 min | 10 min | 30 min | ~45 min |
| Medium (1-100M) | 2 min | 30 min | 15 min | 45 min | 2 hr | ~3.5 hr |
| Large (>100M) | 10 min | 2 hr | 1 hr | 3 hr | 8 hr | ~14 hr |

Behavioral validation (Time-Travel Testing) is the longest phase but provides the strongest guarantee.

---

### Skipping Validation (Not Recommended)

In exceptional circumstances, validation can be skipped:

```bash
curl -X POST https://api.sensei.ai/v1/migrations/{id}/skip-validation \
  -d '{"reason": "Emergency cutover - will validate post-production"}'
```

**Warning:** Skipping validation:
- Produces no Behavioral Equivalence Certificate
- Voids any compliance guarantees
- Should only be done with documented justification

→ [Behavioral Equivalence Certification](../components/kora/behavioral-equivalence-certification.md)
→ [Time-Travel Testing](../components/kora/time-travel-testing.md)
→ [Operations Overview](README.md)
