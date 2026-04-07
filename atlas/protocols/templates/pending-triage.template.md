# Pending Triage Template
# Location: departments/overview/reports/{feature}/pending-triage.md
# APPEND-ONLY — never overwrite or restructure existing entries.
# Written by ALL D04 agents when encountering inconsistencies.
# Read and classified by Overview Agent (triggered by Team Lead every 3 iterations).

---

## entry-{N}
reporter: {devId} | task: {TASK-ID} | timestamp: {ISO8601}
severity: MINOR | MAJOR | CRITICAL
type: file-scope-request | missing-contract | spec-ambiguity | api-change | other
description: >
  {Clear description of the inconsistency or request. One paragraph max.
  E.g.: "Need to create src/Domain/Auth/EmailValidator.cs but this file
  is not in my declared file-scope."}
status: OPEN
resolution: [populated by Overview Agent]

---

## Entry Types

| Type | When to Use | Default Severity |
|---|---|---|
| file-scope-request | Dev needs to create a file outside their declared file-scope | MINOR |
| missing-contract | A required contract file does not exist for a declared dependency | MAJOR |
| spec-ambiguity | The task spec is ambiguous and Dev cannot proceed without clarification | MAJOR |
| api-change | Dev changed a public API that already has a contract file (must version contract) | MINOR |
| other | Any other inconsistency not covered by above types | MAJOR |

## Severity Definitions

| Severity | Overview Agent Action | Team Lead Action |
|---|---|---|
| MINOR | Auto-resolve: mark RESOLVED, dev proceeds on next iteration | None — Overview handles |
| MAJOR | Escalate to Team Lead with classification and suggested resolution | Route fix instructions or approve workaround |
| CRITICAL | Escalate to Team Lead immediately. Team Lead may pause D04 and route to D00 | Evaluate: pause D04 or route to D00 |

## Usage Rules

1. **Append only** — each new entry gets the next sequential number (entry-1, entry-2, ...).
2. **Never edit** existing entries — only Overview Agent may populate the `resolution` field.
3. **Never wait** — Dev appends and immediately continues with non-blocked tasks.
4. **One file per feature** — all D04 agents for a given feature write to the same file.
5. The `status` field is always `OPEN` when written by a Dev. Overview Agent changes it to `RESOLVED` or `ESCALATED`.
