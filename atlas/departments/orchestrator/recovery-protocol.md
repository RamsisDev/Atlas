# Recovery Protocol — D00 Failure Handling & Examples

## Department Checkpoint

Every department writes `dept-checkpoint.md` at defined intervals (see `protocols/templates/dept-checkpoint.template.md`). D00 reads this file to determine the correct recovery strategy — never guess; always read the checkpoint first.

The `rollback-point` in the checkpoint is a git commit SHA. If recovery requires rollback, instruct the next department instance to reset to that SHA before resuming.

## Failure Types and Recovery Actions

| Failure Type | Detection | Action | Data Preserved |
|---|---|---|---|
| **Agent crash** (no state written) | No ralph-state at expected time | Re-spawn from last dept-checkpoint. All committed artifacts intact. | All committed artifacts |
| **D05 CRITICAL finding** | Security verdict: CRITICAL | Targeted fix-request for affected code only. Unaffected code preserved. | All unaffected code |
| **Gate rejection** after partial work | Gate B REJECTED after D04 started | Preserve D04 work in staging branch; re-enter D01 with rejection feedback. | Work preserved in branch |
| **Repeated gate failures** | Retry count ≥ max (default 2) | BLOCKED — surface to user with full history and accumulated feedback. | All artifacts + feedback history |

## Retry Rules

| Department | Max Retries | Trigger |
|---|---|---|
| D01 | 2 | Gate A FAIL or Gate B REJECTED |
| D02 | 0 | No retries — REJECTED routes back to D01 |
| D03 | 0 | Gate C TIMEOUT → BLOCKED immediately |
| D04 | — | PARTIAL tasks re-queued by Team Lead internally |
| D05 | 2 fix cycles | REQUIRES_FIX findings after re-verify |
| D06 | 0 | Failure → BLOCKED |

## BLOCKED State

Trigger: retry limit exceeded, Gate C timeout, Gate D security escalation, or D06 failure.

Steps:
1. Write `pipeline-state.md` with `status: BLOCKED`.
2. Add an entry to `blockers[]`: id, stage, description, file path of the relevant artifact.
3. Surface to user: _"Pipeline BLOCKED at {stage}. See `pipeline-state.md` → blockers. Resolve and reply RESUME to continue."_
4. Stop. Do not attempt to continue autonomously.

On RESUME signal from user:
- Read `pipeline-state.md`. Re-read `blockers[]` to confirm resolution.
- If blocker is resolved → clear the entry, set `status: RUNNING`, continue from `current_stage`.
- If blocker is not resolved → re-surface and stop again.

## FAILED State

Only set `status: FAILED` if a department produces an unrecoverable error: crash, missing output after all retries, or corrupted state file.

Steps:
1. Write `pipeline-state.md` with `status: FAILED` and a `failure_reason` field.
2. Do not attempt recovery. Report to user with the exact failure and file path.

## Recovery on Re-spawn (Ralph Checkpoint)

1. Read `pipeline-state.md`.
2. `status: RUNNING` → resume from `current_stage`. Do not repeat completed stages.
3. `status: BLOCKED` → re-surface the blocker and wait.
4. `status: COMPLETE` or `FAILED` → do not re-run. Report the final status only.

---

## Examples

<example>
<input>Feature request: "Add JWT authentication with refresh tokens"</input>
<action>
1. slug = jwt-auth. Create pipeline-state.md: status=RUNNING, stage=D01, iteration=1.
2. Spawn D01 General Analyst → openspec/specs/jwt-auth-spec.md produced.
3. Spawn Gate Validator → openspec/changes/jwt-auth-gate-a.md → PASS.
4. Update pipeline-state.md: stage=D02, iteration=2.
5. Spawn D02 agents in parallel → openspec/changes/jwt-auth-verdict.md → APPROVED.
6. Update pipeline-state.md: stage=D03, iteration=3.
7. Spawn D03 → openspec/changes/jwt-auth-master-plan.md produced.
8. Surface to user: "Gate C — review jwt-auth-master-plan.md and reply APPROVED."
9. User replies APPROVED → stage=D04, iteration=4. Continue.
</action>
</example>

<example>
<input>Pipeline resumed after RALPH_CHECKPOINT. pipeline-state.md: stage=D04, iteration=6.</input>
<action>
1. Read pipeline-state.md: stage=D04, iteration=6, status=RUNNING.
2. D01, D02, D03 are complete — do not re-run them.
3. Read D04 Team Lead state file to check current task completion.
4. Resume D04 coordination from where Team Lead left off.
</action>
</example>

<example>
<input>Gate A failed twice. D01.retries = 2.</input>
<action>
1. Do not spawn D01 again.
2. Write pipeline-state.md: status=BLOCKED, blockers=[{id: BLK-001, stage: Gate A, description: "Spec failed consistency check twice. Manual review required.", file: "openspec/specs/{slug}-spec.md"}].
3. Surface to user: "Pipeline BLOCKED. Spec failed Gate A twice. Review openspec/specs/{slug}-spec.md, correct ambiguities, and reply RESUME."
4. Stop.
</action>
</example>
