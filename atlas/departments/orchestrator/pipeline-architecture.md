# Pipeline Architecture — D00 Reference

## Flow

```
User Feature Request
 └─► D00 Pipeline Orchestrator
      ├─► D01 Analysis & Spec Design    → Gate A: consistency check
      ├─► D02 Viability Review          → Gate B: APPROVED / REJECTED
      ├─► D03 Project Management        → Gate C: human approval (with timeout)
      ├─► D04 Tech Execution ◄──► Overview Agent (pull-based, every 3 iters)
      │    ├─ Team Lead
      │    ├─ Dev Agents × N (parallel batches)
      │    ├─ QA Agent (parallel groups)
      │    ├─ Software Architect Agent
      │    └─ Integration Agent
      ├─► D05 Security Audit            → Gate D: Team Lead sign-off
      └─► D06 Reconciliation            → Gate E: security clearance
           └─► Constitution Steward (batches shared-proposals/)
```

## Five Gates

| Gate | After | Condition | On Fail |
|---|---|---|---|
| A | D01 | Spec internally consistent, no ambiguity | Retry D01 (max 2) |
| B | D02 | APPROVED / REJECTED | REJECTED → return to D01 (counts as D01 retry) |
| C | D03 | Explicit human sign-off | TIMEOUT → BLOCKED |
| D | D05 | Security Lead clears all findings | Fix cycle → re-verify (max 2 cycles) |
| E | D06 | Spec and code synchronized | Pipeline incomplete → BLOCKED |

## Department Responsibilities

| Dept | Name | Agents | Spawns |
|---|---|---|---|
| D01 | Analysis & Spec Design | General Analyst, Scout, Spec Writer × N | Sequentially |
| D02 | Viability Review | Spec Verifier, Architecture Analyst, Dependency Checker, Security Pre-scanner | In parallel |
| D03 | Project Management | Project Manager, Task Planner, Reconciliation Analyst | Sequentially |
| D04 | Tech Execution | Team Lead → Dev Agents, QA, Software Architect, Integration Agent | Team Lead manages |
| D05 | Security Audit | Security Lead, Change Auditor, Prompt Injection Scanner, Architecture Compliance | Security Lead manages |
| D06 | Reconciliation | Reconciliation Lead, Docs Auditor, Spec Consolidator | Sequentially |

## Multi-Feature Isolation

Each feature runs on its own git branch. Shared documents are read from `main` and modified through `shared-proposals/`. No two features write the same file in the same parallel group — guaranteed by Gate C file-scope check.
