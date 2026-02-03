# Migration Lifecycle

## Phases, States, and Transitions

Every Sensei migration follows a predictable lifecycle. Understanding this lifecycle helps you plan operations, set expectations with stakeholders, and troubleshoot issues.

---

### The Five Phases

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Phase 1      Phase 2        Phase 3        Phase 4       Phase 5      │
│  PLANNING     APPROVAL       EXECUTION      VALIDATION    CERTIFICATION│
│                                                                         │
│  ┌────────┐   ┌────────┐    ┌──────────┐   ┌──────────┐  ┌──────────┐ │
│  │Analyze │   │ Review │    │ Execute  │   │ Verify   │  │ Certify  │ │
│  │Schema  │ → │ Plan   │ →  │ Migration│ → │ Results  │→ │ Complete │ │
│  │Generate│   │ Approve│    │ Monitor  │   │ Fix if   │  │ Handoff  │ │
│  │Plan    │   │        │    │          │   │ needed   │  │          │ │
│  └────────┘   └────────┘    └──────────┘   └──────────┘  └──────────┘ │
│                                                                         │
│  ~1-4 hours   ~1-24 hours   ~hours-days    ~2-8 hours    ~30 minutes  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Phase 1: Planning

**Duration:** 1-4 hours (automated)

**Activities:**
1. **Schema Analysis** — Scout Agents connect to source and target, extract metadata
2. **Statistical Profiling** — MABA computes distributional fingerprints
3. **Semantic Analysis** — LLM inference determines column meanings
4. **Pattern Matching** — Query pattern library for similar prior migrations
5. **Plan Generation** — Architect Agents create migration DAG and transformation code

**States:** `created` → `analyzing` → `planning` → `awaiting_approval`

**Outputs:**
- Complete source schema documentation
- Suggested mappings with confidence scores
- Migration DAG (execution order)
- Risk assessment
- Time and resource estimates

**User actions required:** None (automated). User may be notified of low-confidence mappings.

---

### Phase 2: Approval

**Duration:** Variable (depends on review process)

**Activities:**
1. **Review mappings** — Stakeholders examine suggested mappings
2. **Adjust as needed** — Modify mappings, transformations, or strategy
3. **Configure options** — Set error thresholds, notification preferences
4. **Approve plan** — Authorize execution to begin

**States:** `awaiting_approval` → `approved`

**Outputs:**
- Approved migration plan
- Configuration settings locked
- Audit trail of approvals

**User actions required:**
- Review and approve mappings
- Configure execution options
- Grant final approval

---

### Phase 3: Execution

**Duration:** Hours to days (depends on data volume)

**Activities:**
1. **Data transfer** — Worker Agents read, transform, and write data
2. **Inline validation** — Validator Agents check each batch
3. **Error recovery** — Automatic recovery for known error patterns
4. **Progress reporting** — Real-time updates via AMANI
5. **Checkpointing** — Regular state snapshots for resume capability

**States:** `approved` → `running` → (`paused`) → `completed`

**Outputs:**
- Data migrated to target
- Execution logs
- Error log with recovery outcomes
- Performance metrics

**User actions required:**
- Monitor progress (optional — AMANI alerts on issues)
- Resolve escalated errors
- Approve continuation after major milestones (if configured)

---

### Phase 4: Validation

**Duration:** 2-8 hours (automated)

**Activities:**
1. **Structural validation** — Schema comparison
2. **Statistical validation** — Distribution comparison
3. **Referential validation** — FK integrity verification
4. **Semantic validation** — Business rule checking
5. **Behavioral validation** — Time-Travel Testing

**States:** `completed` → `validating` → `certified` (or `failed`)

**Outputs:**
- Validation results (five perspectives)
- Divergence reports (if any)
- Behavioral Equivalence Certificate (if passed)

**User actions required:**
- Review validation results
- Investigate flagged divergences
- Accept or reject certification

---

### Phase 5: Certification

**Duration:** ~30 minutes (mostly automated)

**Activities:**
1. **Evidence assembly** — Compile all validation evidence
2. **Certificate generation** — Create cryptographically signed certificate
3. **Handoff** — Deliver certificate and documentation to stakeholders
4. **Cleanup** — Archive temporary data, release resources

**States:** `certified` → (terminal)

**Outputs:**
- Behavioral Equivalence Certificate
- Migration summary report
- Archived logs and evidence

**User actions required:**
- Acknowledge completion
- Distribute certificate to stakeholders
- Initiate cutover (if not already done)

---

### State Diagram

```
                    ┌──────────────┐
                    │   created    │
                    └──────┬───────┘
                           │ start
                           ↓
                    ┌──────────────┐
              ┌─────│  analyzing   │─────┐
              │     └──────┬───────┘     │
              │            │ complete    │ fail
              │            ↓             │
              │     ┌──────────────┐     │
              │     │   planning   │─────┤
              │     └──────┬───────┘     │
              │            │ complete    │
              │            ↓             │
              │     ┌──────────────┐     │
   cancel ────┼─────│awaiting_     │     │
              │     │  approval    │     │
              │     └──────┬───────┘     │
              │            │ approve     │
              │            ↓             │
              │     ┌──────────────┐     │
              │     │   approved   │     │
              │     └──────┬───────┘     │
              │            │ execute     │
              │            ↓             │
              │     ┌──────────────┐     │
              │  ┌──│   running    │──┐  │
              │  │  └──────┬───────┘  │  │
              │  │pause    │complete  │fail
              │  ↓         │          │  │
              │ ┌──────┐   │          │  │
              │ │paused│───┘          │  │
              │ └──────┘  resume      │  │
              │            │          │  │
              │            ↓          ↓  ↓
              │     ┌──────────────┐ ┌──────┐
              │     │  completed   │ │failed│
              │     └──────┬───────┘ └──────┘
              │            │ validate    ↑
              │            ↓             │
              │     ┌──────────────┐     │
              │     │  validating  │─────┘
              │     └──────┬───────┘ fail
              │            │ certify
              │            ↓
              │     ┌──────────────┐
              └────→│  cancelled   │
                    └──────────────┘
                           ↑
                    ┌──────────────┐
                    │  certified   │
                    └──────────────┘
```

---

### Duration Estimates

| Phase | Small (<1M rows) | Medium (1-100M) | Large (>100M) |
|-------|-----------------|-----------------|---------------|
| Planning | 30 min | 1-2 hours | 2-4 hours |
| Approval | 1-4 hours | 4-24 hours | 24-72 hours |
| Execution | 1-4 hours | 4-24 hours | 24-168 hours |
| Validation | 1-2 hours | 2-6 hours | 6-24 hours |
| Certification | 15 min | 30 min | 1 hour |

Total elapsed time varies significantly based on approval process duration and migration complexity.

→ [Planning Phase](planning-phase.md)
→ [Execution Phase](execution-phase.md)
→ [Validation Phase](validation-phase.md)
