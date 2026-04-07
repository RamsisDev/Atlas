# pipeline-state.md
# Single source of truth for pipeline position.
# D00 reads and writes this file at every gate transition.

---

```yaml
feature: {feature-slug}
branch: feature/{feature-slug}
current-department: D01
current-gate: GA
gate-status: PENDING       # PENDING | PASSED | APPROVED | REJECTED | BLOCKED | PAUSED
started: {ISO-8601}

history:
  - dept: D01 | gate: GA | status: PASSED | duration: {Nmin}
  - dept: D02 | gate: GB | status: PENDING

retries:
  - dept: D01 | reason: {consistency-check-failed} | count: 1

max-retries-per-gate: 2

# Gate C configuration
gate-c-timeout-hours: 24
gate-c-pending-since: null       # Set when D03 completes
gate-c-reminder-sent: false
gate-c-auto-approve: false       # Explicit opt-in to skip human gate. Never a default.
gate-c-status: PENDING           # PENDING | APPROVED | PAUSED | TIMEOUT

# Checkpoint tracking
last-checkpoint: {ISO-8601}
checkpoint-count: 0
```

---

## Gate Advancement Rules

D00 follows a deterministic state machine. No ambiguity allowed.

| Gate Verdict | Condition | D00 Action |
|---|---|---|
| **PASSED / APPROVED** | Gate validation succeeds | Update history. Advance current-department to next. Write dept-checkpoint.md. Spawn next department. |
| **REJECTED** | retries < max-retries | Increment retry counter. Re-invoke previous department with rejection feedback file as additional context. |
| **REJECTED** | retries ≥ max-retries | Write status: BLOCKED. Surface to user with full retry history and all accumulated rejection feedback. |
| **PENDING** | Gate C only | Record gate-c-pending-since timestamp. Monitor timeout. If timeout exceeded and gate-c-reminder-sent is false: send reminder, enter PAUSED state. If gate-c-auto-approve is true: advance immediately. |
| **FAILED** | Department crash | Read dept-checkpoint.md for last known good state. Re-spawn department from checkpoint. If no checkpoint exists: re-run department from scratch. |

---

## Gate C Timeout Protocol

1. When D03 completes and master-plan.md is ready: D00 sets `gate-c-pending-since` and presents the plan to the user.
2. D00 checks elapsed time at each iteration. If elapsed > `gate-c-timeout-hours` and `gate-c-reminder-sent` is false: write reminder notification and set `gate-c-reminder-sent: true`.
3. D00 enters PAUSED state. The pipeline waits. D00 does NOT auto-approve and does NOT auto-reject.
4. Opt-in auto-approve: for overnight/autonomous runs, the user sets `gate-c-auto-approve: true` before pipeline starts. This is an explicit opt-in documented as a conscious decision, never a silent default.

---

## Multi-Feature Branch Isolation

- D00 creates `feature/{feature-slug}` branch at pipeline start.
- All departments for this feature work exclusively on this branch.
- Shared documents (`constitution.md`, `architecture/overview.md`, `specs/`) are read from `main` at the start of D01. Never modified in-flight.
- D06 creates proposal files in `shared-proposals/` instead of modifying shared documents directly.
- Conflicts between concurrent features surface at merge time, not during pipeline execution.

---

## Shared Proposals Structure

```
shared-proposals/
├── constitution/
│   └── {date}-{feature-slug}.md    # Proposes changes to constitution.md
├── architecture/
│   └── {date}-{feature-slug}.md    # Proposes changes to architecture/overview.md
└── specs/
    └── {date}-{feature-slug}.md    # Proposes new entries to shared specs/
```

The Constitution Steward reads all proposal files after active features complete D06, groups them by target document, applies non-conflicting proposals in a single pass, and surfaces conflicting proposals to the user.
