# Cutover Strategies

## Go-Live Approaches

Cutover is the moment when applications switch from source to target database. The right strategy depends on your downtime tolerance, data consistency requirements, and rollback needs.

---

### Strategy Comparison

| Strategy       | Downtime | Complexity | Rollback    | Best For                          |
| -------------- | -------- | ---------- | ----------- | --------------------------------- |
| **Big Bang**   | Hours    | Low        | Difficult   | Small datasets, planned outages   |
| **Blue-Green** | Minutes  | Medium     | Easy        | Applications with load balancers  |
| **Canary**     | Zero     | High       | Easy        | Large user bases, gradual rollout |
| **Strangler**  | Zero     | High       | Per-feature | Monolith to microservices         |

---

### Big Bang Cutover

All traffic switches at once during a planned outage.

```text
┌─────────────────────────────────────────────────────────────────┐
│                      BIG BANG CUTOVER                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Before:       App ──→ Source DB                                │
│                                                                  │
│  Cutover:      App ──× (offline)                                │
│                Sensei: Final sync + validation                  │
│                                                                  │
│  After:        App ──→ Target DB                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Steps:**

1. Schedule maintenance window
2. Stop application traffic
3. Final incremental sync
4. Validate target
5. Update connection strings
6. Start application traffic
7. Monitor

**Timeline:**

```text
T-0:00  Stop application
T-0:05  Final sync begins
T-0:15  Final sync complete
T-0:20  Validation complete
T-0:25  Connection strings updated
T-0:30  Application restarted
T-0:35  Monitoring confirms success

Total downtime: ~35 minutes
```

---

### Blue-Green Cutover

Two environments; switch traffic with load balancer.

```text
┌─────────────────────────────────────────────────────────────────┐
│                     BLUE-GREEN CUTOVER                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Before:                                                         │
│            ┌─────────┐                                          │
│  Traffic → │ Blue    │ ──→ Source DB (active)                   │
│            │ (App)   │                                          │
│            └─────────┘                                          │
│            ┌─────────┐                                          │
│            │ Green   │ ──→ Target DB (standby)                  │
│            │ (App)   │                                          │
│            └─────────┘                                          │
│                                                                  │
│  Cutover: Load balancer switches traffic Blue → Green           │
│                                                                  │
│  After:                                                          │
│            ┌─────────┐                                          │
│            │ Blue    │ ──→ Source DB (standby/rollback)        │
│            │ (App)   │                                          │
│            └─────────┘                                          │
│            ┌─────────┐                                          │
│  Traffic → │ Green   │ ──→ Target DB (active)                   │
│            │ (App)   │                                          │
│            └─────────┘                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Steps:**

1. Deploy application to Green pointing to Target
2. Run incremental sync until Green is current
3. Validate Green environment (smoke tests)
4. Switch load balancer: Blue → Green
5. Monitor Green
6. Keep Blue ready for rollback
7. Decommission Blue after confidence period

**Rollback:** Switch load balancer back to Blue.

---

### Canary Cutover

Gradually shift traffic from source to target.

```text
┌─────────────────────────────────────────────────────────────────┐
│                      CANARY CUTOVER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Phase 1: 5% traffic to Target                                  │
│           │████░░░░░░░░░░░░░░░░░│                              │
│           Source (95%)  Target (5%)                             │
│                                                                  │
│  Phase 2: 25% traffic to Target                                 │
│           │████████████░░░░░░░░░│                              │
│           Source (75%)  Target (25%)                            │
│                                                                  │
│  Phase 3: 50% traffic to Target                                 │
│           │██████████████████░░░│                              │
│           Source (50%)  Target (50%)                            │
│                                                                  │
│  Phase 4: 100% traffic to Target                                │
│           │████████████████████│                               │
│           Target (100%)                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Steps:**

1. Set up traffic splitting (load balancer, feature flags)
2. Route 5% of traffic to Target
3. Monitor error rates, latency, user feedback
4. If healthy, increase to 25%
5. Continue increasing until 100%
6. Decommission source after confidence period

**Rollback:** Reduce Target percentage back to 0%.

---

### Strangler Pattern

Migrate feature-by-feature rather than database-by-database.

```text
┌─────────────────────────────────────────────────────────────────┐
│                     STRANGLER PATTERN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Monolith                           Microservices               │
│  ┌──────────────┐                  ┌──────────────┐             │
│  │ Users        │ ─── migrate ───→ │ User Service │             │
│  │ Orders       │                  │              │             │
│  │ Products     │                  └──────────────┘             │
│  │ Payments     │                                               │
│  │ Reports      │                  ┌──────────────┐             │
│  └──────────────┘ ─── migrate ───→ │ Order Service│             │
│         │                          │              │             │
│         │                          └──────────────┘             │
│         ↓                                                        │
│  Gradually shrink monolith as services are extracted            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Steps:**

1. Identify bounded contexts
2. Extract one context (e.g., Users)
3. Migrate User data to new service
4. Update monolith to call User service
5. Repeat for other contexts

---

### Pre-Cutover Checklist

```yaml
pre_cutover_checklist:
  technical:
    - [ ] Migration validated (Behavioral Equivalence Certificate)
    - [ ] Incremental sync lag < acceptable threshold
    - [ ] Target performance tested under load
    - [ ] Connection strings/endpoints prepared
    - [ ] Rollback procedure documented and tested
    - [ ] Monitoring and alerts configured

  operational:
    - [ ] Maintenance window scheduled and communicated
    - [ ] Stakeholders notified
    - [ ] Support team briefed
    - [ ] Rollback criteria defined
    - [ ] Success criteria defined

  business:
    - [ ] User communication sent
    - [ ] Customer support prepared
    - [ ] Business continuity plan reviewed
```

---

### Cutover Runbook Template

```markdown
# Cutover Runbook: [Migration Name]

## Pre-Cutover (T-24 hours)

- [ ] Verify final incremental sync status
- [ ] Confirm maintenance window with stakeholders
- [ ] Test rollback procedure

## Cutover Execution

T-0:00 Announce cutover starting
T-0:05 [Action: Stop application traffic]
T-0:10 [Action: Verify traffic stopped]
T-0:15 [Action: Final sync]
T-0:25 [Action: Validation checks]
T-0:30 [Action: Switch connection strings]
T-0:35 [Action: Start application]
T-0:40 [Action: Smoke tests]
T-0:45 Announce cutover complete

## Rollback Triggers

- Error rate > 5%
- Latency > 2x baseline
- Critical functionality broken
- Data integrity issues detected

## Rollback Procedure

1. [Step 1]
2. [Step 2]
3. [Step 3]
```

→ [Rollback](rollback.md)
→ [Incremental Migrations](incremental-migrations.md)
→ [Post-Migration](post-migration.md)
