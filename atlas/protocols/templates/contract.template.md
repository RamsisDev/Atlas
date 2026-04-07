# Interface Contract Template
# Location: departments/tech/work/{feature}/contracts/{ClassName}.contract.md
# Written by Dev Agents when completing tasks that expose public APIs.
# IMMUTABLE once written. Version as {ClassName}.v2.contract.md if API changes.

---

# contracts/{ClassName}.contract.md

author: {devId} | task: {TASK-ID} | timestamp: {ISO8601}

## public-api

{EntityOrInterface}:
  properties:
    - {PropertyName}: {Type} ({constraints, e.g. unique, validated})
    - {PropertyName}: {Type}
  methods:
    - {static?} {MethodName}({param}: {Type}): {ReturnType}
    - {MethodName}({param}: {Type}): {ReturnType}

## location
file: {relative/path/to/source/file}

## notes
{Any constraints, design decisions, or usage warnings downstream devs need to know.}
{E.g.: "Email uniqueness enforced at repository level, not entity."}
{E.g.: "PasswordHash uses BCrypt with work factor 12."}

---

## Contract Protocol Rules

| # | Rule | Detail |
|---|---|---|
| 1 | Write on public API completion | When a Dev completes a task that exposes a public interface, public class methods, or API endpoints, it writes a contract file to departments/tech/work/{feature}/contracts/ |
| 2 | Read before dependent task | When a Dev starts a task with dependencies (deps field in master-plan.md), it reads contracts/ for all dependency tasks before beginning implementation |
| 3 | Missing contract escalation | If a contract file is missing for a required dependency, the Dev appends to pending-triage.md as MAJOR and skips to a non-blocked task |
| 4 | Contract immutability | Once written, a contract is never modified. If the API changes during implementation, the Dev writes a new versioned contract ({ClassName}.v2.contract.md) and appends to pending-triage.md |
| 5 | Integration Agent validates | After all groups complete, the Integration Agent compares all contract files against the actual implementation to detect drift |
