# Planning Phase

## Pre-Migration Preparation and Analysis

The planning phase is where Sensei analyzes your source and target systems, understands the data, and generates a migration plan. This phase is largely automated but produces outputs that require human review.

---

### What Happens

```text
Source Connection              Target Connection
      │                              │
      ↓                              ↓
┌─────────────────┐          ┌─────────────────┐
│ Schema          │          │ Schema          │
│ Extraction      │          │ Definition      │
└────────┬────────┘          └────────┬────────┘
         │                            │
         ↓                            ↓
┌─────────────────────────────────────────────┐
│            SCHEMA ANALYSIS                   │
│  • Structural analysis (tables, columns)    │
│  • Statistical profiling (distributions)    │
│  • Semantic inference (business meaning)    │
└─────────────────────┬───────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────┐
│            PATTERN MATCHING                  │
│  • Query pattern library                    │
│  • Find similar prior migrations            │
│  • Apply learned mappings                   │
└─────────────────────┬───────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────┐
│            PLAN GENERATION                   │
│  • Generate mappings (with confidence)      │
│  • Create migration DAG                     │
│  • Write transformation code                │
│  • Estimate time and resources              │
└─────────────────────┬───────────────────────┘
                      │
                      ↓
                 Migration Plan
```

---

### Timeline

| Step                  | Duration      | Resource Usage           |
| --------------------- | ------------- | ------------------------ |
| Schema extraction     | 1-10 minutes  | Low (metadata only)      |
| Statistical profiling | 10-60 minutes | Medium (samples data)    |
| Semantic inference    | 10-30 minutes | High (LLM inference)     |
| Pattern matching      | 1-5 minutes   | Low (vector query)       |
| Plan generation       | 5-15 minutes  | Medium (code generation) |

**Total:** 30 minutes to 4 hours depending on schema complexity.

---

### Planning Configuration

Configure how aggressively Sensei analyzes the source:

```yaml
planning:
  # Statistical profiling depth
  profiling:
    sample_rate: 0.1 # 10% sample for statistics
    max_sample_rows: 100000 # Cap at 100K rows per table
    compute_histograms: true
    detect_patterns: true # Regex pattern detection

  # Semantic analysis
  semantic:
    infer_business_meaning: true
    detect_pii: true
    use_pattern_library: true
    min_confidence_threshold: 0.7

  # Plan generation
  generation:
    auto_approve_above: 0.95 # Auto-approve high-confidence
    flag_below: 0.8 # Flag low-confidence for review
    generate_rollback_plan: true
```

---

### Planning Outputs

#### 1. Source Schema Documentation

Complete documentation of the source system:

```yaml
source_schema:
  tables: 247
  total_columns: 3891
  total_rows: 52000000
  relationships:
    explicit_fks: 189
    inferred_relationships: 47

  table_details:
    - name: customers
      columns: 23
      rows: 1250000
      primary_key: customer_id
      foreign_keys:
        - column: region_id
          references: regions.id
      sensitive_columns:
        - name: email (PII: email)
        - name: ssn_encrypted (PII: SSN)
```

#### 2. Mapping Recommendations

For each source column:

```yaml
mappings:
  - source:
      table: customers
      column: cust_ref
      type: VARCHAR(15)
    target:
      table: customers
      column: customer_reference
      type: VARCHAR(20)
    confidence: 0.94
    transformation: 'CAST(source.cust_ref AS VARCHAR(20))'
    reasoning: 'Name similarity (0.85) + type compatibility (1.0) + pattern match to prior migration mig_xyz (0.97)'
    flags: []

  - source:
      table: orders
      column: dt_created
      type: VARCHAR(19)
    target:
      table: orders
      column: created_at
      type: TIMESTAMP
    confidence: 0.78
    transformation: "TO_TIMESTAMP(source.dt_created, 'YYYY-MM-DD HH24:MI:SS')"
    reasoning: 'Semantic inference (date) + pattern detection (ISO 8601)'
    flags:
      - 'Low confidence - verify date format'
      - '12 records have non-conforming format'
```

#### 3. Migration DAG

Execution order respecting dependencies:

```yaml
execution_plan:
  phases:
    - phase: 1
      name: 'Reference Data'
      tables: [regions, statuses, categories]
      parallel: true
      estimated_duration: 5 minutes

    - phase: 2
      name: 'Customer Data'
      tables: [customers, contacts, addresses]
      parallel: true
      dependencies: [phase_1]
      estimated_duration: 2 hours

    - phase: 3
      name: 'Transaction Data'
      tables: [orders, order_items, payments]
      parallel: false # FK dependencies
      dependencies: [phase_2]
      estimated_duration: 18 hours
```

#### 4. Risk Assessment

```yaml
risk_assessment:
  overall_risk: medium

  risks:
    - type: data_loss
      severity: low
      description: '3 VARCHAR columns will be truncated'
      affected: ['products.description', 'notes.content', 'logs.message']
      mitigation: 'Non-critical fields; truncation acceptable'

    - type: semantic_ambiguity
      severity: medium
      description: '47 columns have <80% mapping confidence'
      affected: [list of columns]
      mitigation: 'Manual review required'

    - type: performance
      severity: low
      description: 'orders table (45M rows) may cause bottleneck'
      mitigation: 'Increase parallel workers for phase 3'
```

#### 5. Resource Estimates

```yaml
estimates:
  duration:
    optimistic: 22 hours
    expected: 28 hours
    pessimistic: 40 hours

  resources:
    peak_workers: 16
    peak_memory_gb: 64
    network_transfer_gb: 127
    estimated_cost: $340
```

---

### Reviewing the Plan

Before approving, review:

1. **Low-confidence mappings** — Verify the suggested mapping makes sense
2. **Flagged transformations** — Confirm data format assumptions
3. **PII handling** — Ensure sensitive data is handled appropriately
4. **Risk items** — Accept or mitigate identified risks
5. **Resource estimates** — Ensure infrastructure can handle the load

→ [Execution Phase](execution-phase.md)
→ [Error Handling](error-handling.md)
→ [Operations Overview](README.md)
