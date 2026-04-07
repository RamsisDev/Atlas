# Team Lead — Full Execution Protocol

Reference file for D04-00 Team Lead. Loaded via context_chain.

---

## Execution Cycle (15 Steps)

```
D04 EXECUTION FLOW:

1.  Team Lead reads master-plan.md + skills-mapping.md
    → Identifies parallel groups (A, B, C, ...) from master-plan.md
    → Assembles skill file list per dev from skills-mapping.md

2.  Spawn Software Architect (always first, before any dev)
    → Provides: constitution.md, master-plan.md, architecture/ folder
    → Wait for architecture-review.md with status: APPROVED | APPROVED_WITH_NOTES | BLOCKED

3.  If BLOCKED:
    → Extract remediation instructions from architecture-review.md
    → Route to D00 with instructions for plan revision
    → HALT D04 — do NOT spawn any dev agents

4.  spawn_batch(Group A tasks)
    Each dev receives spawn context:
      - constitution.md                    (always)
      - master-plan.md                     (always)
      - dev-tasks.md#{devId}              (only this dev's tasks)
      - contracts/                         (if deps exist)
      - skill files from skills-mapping    (from skills-mapping.md entry for devId)
      - architecture-review.md            (always — from Software Architect)
      - ralph-state/{devId}/              (if resuming a PARTIAL)

5.  Wait for all Group A devs → status: COMPLETE or PARTIAL

6.  For each PARTIAL dev:
    a. Count items in ## pending section of ralph-state
    b. If 1-2 items: EXTEND — re-spawn dev with additional iterations
       Dev reads NEXT field from ralph-state and continues
    c. If 3+ items: OVERFLOW — write overflow task to
       departments/pm/work/{feature}/overflow-tasks/overflow-{N}.md
       Queue for second D04 pass after current batch completes

7.  Overview Agent checkpoint (after Group A, and every 3 Ralph iterations thereafter)
    → Reads departments/overview/reports/{feature}/pending-triage.md
    → Classifies OPEN entries: MINOR (auto-resolve) | MAJOR (escalate) | CRITICAL (halt)
    → For CRITICAL: route to D00, pause D04

8.  spawn_batch(Group B tasks)
    → Each Group B dev reads contracts/ folder FIRST before implementing
    → Missing contract for a declared dep → dev appends MAJOR to pending-triage.md,
      skips that dependency, continues with non-blocked work

9.  Repeat steps 6-8 for Groups C, D, ... until all groups complete

10. Spawn Integration Agent (after FINAL group completes)
    → Provides: all contracts/, all delta-specs/, master-plan.md reconciliation-strategy
    → Wait for integration-report.md with status: PASS | FAIL

11. If Integration FAIL:
    → Extract per-dev fix instructions from ## issues section
    → Route specific instructions to responsible devs
    → Re-spawn targeted devs (counts against their iteration budget)
    → Re-spawn Integration Agent to re-validate
    → integration-report.md must be PASS before proceeding

12. Team Lead sign-off
    → Verify: all delta-specs/ exist for all devs
    → Verify: all tasks with public API have contract files
    → Verify: integration-report.md status = PASS
    → Verify: no OPEN CRITICAL entries in pending-triage.md

13. Spawn QA Agent
    → Provides: dev-tasks.md, qa-tasks.md, all delta-specs/, constitution.md
    → QA classifies test scenarios into parallel QA groups
    → QA spawns QA sub-agents via spawn_batch() per group
    → QA Root compiles qa-results.md

14. Review qa-results.md
    → All scenarios must PASS
    → Any FAIL: route fix to responsible dev, re-test that scenario only

15. Gate D validation (signal D00):
    → All tests PASS in qa-results.md
    → Team Lead sign-off complete
    → All delta-specs updated (no TODO entries)
    → No OPEN fix-requests in pending-triage.md
    → integration-report.md = PASS
```

---

## Skill Context Assembly per Dev

```javascript
// Team Lead assembles spawn context per dev from skills-mapping.md
const devContext = [
  read('constitution.md'),               // always
  read('master-plan.md'),                // always
  read(`dev-tasks.md#${devId}`),        // only this dev's tasks
  read('contracts/'),                    // if deps exist for this dev
  ...skillFiles(devId),                  // from skills-mapping.md
  read(`architecture-review.md`),        // always
  read(`ralph-state/${devId}/`),         // if resuming
];

// skillFiles() reads skills-mapping.md, filters to devId,
// returns the list of skill file paths for that dev's tasks.
// Token budget is pre-calculated in skills-mapping.md.
```

---

## PARTIAL Decision Table

| Remaining items in ## pending | Action |
|---|---|
| 0 | COMPLETE — no action needed |
| 1–2 | EXTEND — re-spawn dev with current ralph-state as context |
| 3+ | OVERFLOW — write overflow-{N}.md, queue for second pass |

---

## Overview Agent Trigger Points

- After Group A completion (always)
- After every 3 Ralph iterations (Team Lead counts)
- After Integration Agent if FAIL

The Overview Agent reads `departments/overview/reports/{feature}/pending-triage.md`
and resolves OPEN entries or escalates them. Team Lead resumes after Overview completes.
