# Architecture Review Template
# Location: departments/tech/work/{feature}/architecture-review.md
# Written by Software Architect (D04-02) before any Dev Agent is spawned.
# Read by all Dev Agents as part of their spawn context.

---

# architecture-review.md | feature: {feature-name}

reviewer: software-architect | timestamp: {ISO8601}
status: APPROVED | APPROVED_WITH_NOTES | BLOCKED

## layer-validation
{For each task, verify the target file path matches the declared layer.}
- {TASK-ID} ({layer}) → PASS | FAIL: {reason if FAIL}
- {TASK-ID} ({layer}) → PASS | FAIL: {reason if FAIL}

## cross-cutting-concerns
{Identify shared services, middleware, and infrastructure components that multiple tasks
depend on but that no single task is responsible for creating.}
- {ConcernName}: {exists at path} | MISSING — no task owns its creation
  {Coordination note if needed.}

## dependency-direction
{Validate that the planned task dependencies respect the project's layer boundaries.
E.g., domain must not depend on infrastructure.}
- All task dependencies follow {layer} → {layer} → {layer}: FOLLOWED | VIOLATED
- {Specific violation if any}: {TASK-ID} → {TASK-ID} violates {rule}

## contract-sufficiency
{Check that every task with downstream dependents has enough specification to produce
a meaningful contract file.}
- {TASK-ID} → has {N} downstream dependents ({TASK-IDs}).
  Spec detail is SUFFICIENT | INSUFFICIENT for contract generation.
  {If INSUFFICIENT: what is missing.}

## notes-for-devs
{Precise, actionable guidance for Dev Agents. Devs read this section before implementing.}
- {TASK-ID}: {specific architectural guidance, e.g. "use IAuthRepository interface from
  TASK-03, not direct DbContext access. Ensure contract is read before implementing."}

---

## Status Definitions

| Status | Meaning |
|---|---|
| APPROVED | All tasks pass layer validation, dependency direction, and contract sufficiency checks. |
| APPROVED_WITH_NOTES | Minor concerns identified. Devs must read notes-for-devs. Implementation may proceed. |
| BLOCKED | One or more critical violations found. Team Lead must route to D00 for plan revision before any Dev is spawned. |
