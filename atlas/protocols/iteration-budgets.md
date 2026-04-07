# Iteration Budgets per Agent Type

Every agent has a fixed maximum number of Ralph iterations, calibrated to the expected scope of its work.

**Dev Agent exception:** `max_iterations = task_score + 2` where score is the fractal complexity score from D03 (see `protocols/fractal-scoring.md`).

**EARLY_EXIT_WARNING** is triggered at 80% of budget. The orchestrating parent has two options:
- **Extend by 3 iterations** — default for agents making steady progress. Re-spawn with same task and existing state file.
- **Accept PARTIAL** — create an overflow-task entry for remaining work (see `protocols/fractal-scoring.md`). Use when remaining work is separable and schedulable independently.

Every PARTIAL must result in an explicit decision recorded in the orchestrator's own ralph-state file. Silent PARTIAL ignoring is not allowed.

---

## Budget Table

| Agent | Department | Max Iter. | Rationale |
|---|---|---|---|
| **Pipeline Orchestrator** | D00 | 10 | Must survive full pipeline lifecycle; reads gate files, not complex analysis |
| Gate Validator | D00 | 1 | Single-pass validation; bounded scope |
| **Scout** | D01 | 5 | File discovery is bounded; if more needed, codebase exceeds scope |
| Spec Writer ×N | D01 | 3 | One module per writer; bounded scope |
| Consistency Checker | D01 | 1 | Single-pass validation; REJECTED if ambiguous |
| Domain Expert | D01 | 2 | Reads architecture skills and domain model; validation, not generation |
| **Architecture Analyst** | D02 | 1 | Point review; repeated passes indicate spec too ambiguous — REJECTED |
| Dependency Checker | D02 | 1 | Single-pass dependency graph |
| Security Pre-scanner | D02 | 1 | Single-pass risk analysis |
| Performance Analyst | D02 | 1 | Single-pass bottleneck evaluation; flag, do not fix |
| **Project Manager** | D03 | 2 | Plan generation + revision after human gate feedback |
| Task Planner | D03 | 2 | Decomposition + refinement |
| Reconciliation Analyst | D03 | 2 | Dependency graph analysis + merge strategy |
| Skills Mapper | D03 | 1 | Single-pass mapping of skills to tasks |
| **Team Lead** | D04 | 4 | Integration + review checkpoints across parallel groups |
| Dev Agent ×N | D04 | score+2 | Calibrated by task complexity score |
| QA Root Agent | D04 | 6 | Coordinate QA groups, compile results, re-test failures |
| QA Test Sub-agent | D04 | 2 | Execute test + one retry; bounded scope per scenario |
| Software Architect Agent | D04 | 2 | Architecture review of implementation decisions; read-only + guidance |
| Integration Agent | D04 | 3 | Contract verification + integration testing after each parallel group |
| **Security Lead** | D05 | 8 | Multi-file reading + cross-reference analysis |
| Change Auditor | D05 | 3 | Inventory comparison against spec |
| Dep. Vulnerability Scanner | D05 | 2 | Dependency audit against known vulnerability databases |
| **Reconciliation Lead** | D06 | 5 | Reading all department outputs for final audit |
| Spec Consolidator | D06 | 3 | Delta merge + archive operations |
| Metrics Compiler | D06 | 1 | Single-pass metrics collection from all department artifacts |
| **Overview Agent** | Transversal | 3 | Triage + verification with read-only code access |  
| Escalation Router | Transversal | 1 | Single-pass escalation package assembly |

---

## Recovery Strategies (Department Checkpoint)

D00 determines recovery action based on the failure type detected in `dept-checkpoint.md`:

| Failure Type | Detection | Action | Data Preserved |
|---|---|---|---|
| **Agent crash** (no state written) | No ralph-state at expected time | Re-spawn from last dept-checkpoint. All committed artifacts intact. | All committed artifacts |
| **D05 CRITICAL finding** | Security verdict: CRITICAL | Targeted fix-request for affected code only. Unaffected code preserved. | All unaffected code |
| **Gate rejection** after partial work | Gate B REJECTED after D04 started | Preserve D04 work in staging branch; re-enter at D01 with feedback. | Work preserved in branch |
| **Repeated gate failures** | Retry count exceeds max (default: 2) | BLOCKED status; surface to user with full history and accumulated feedback. | All artifacts + feedback history |
