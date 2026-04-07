# Execution Protocol — D00 Stage-by-Stage

Execute stages in this exact order. Each stage is one action per iteration. Do not batch stages.

---

## Stage 1 — Initialize

- Validate a feature request exists (from user input or `feature-request.txt`).
- Derive `{feature-slug}` from the request (kebab-case, max 30 chars).
- Create `pipeline-state.md`: `status: RUNNING`, `current_stage: D01`, `iteration: 1`.
- Create `openspec/specs/`, `openspec/changes/`, `security/` folders if absent.

---

## Stage 2 — D01: Analysis & Spec Design

- Spawn D01 General Analyst with the feature request and slug as context.
- Expected output: `openspec/specs/{slug}-spec.md`.
- Spawn Gate Validator sub-agent against the spec (see Gate Validator below).
- Gate A PASS → set `current_stage: D02`. Gate A FAIL → increment `D01.retries`. If retries ≥ 2 → BLOCKED.

---

## Stage 3 — D02: Viability Review

- Spawn D02 agents in parallel: Spec Verifier, Architecture Analyst, Dependency Checker, Security Pre-scanner.
- Expected output: `openspec/changes/{slug}-verdict.md`.
- Gate B APPROVED → set `current_stage: D03`.
- Gate B REJECTED → return to D01 with rejection rationale attached. Counts as a D01 retry.

---

## Stage 4 — D03: Project Management

- Spawn D03 Project Manager then Task Planner sequentially.
- Expected outputs: `openspec/changes/{slug}-master-plan.md`, `{slug}-file-scope-check.md`, `{slug}-skills-mapping.md`.
- Surface the master plan to the user: _"Gate C — human approval required. Review `openspec/changes/{slug}-master-plan.md` and reply APPROVED to continue."_
- Gate C APPROVED → set `current_stage: D04`. Gate C TIMEOUT (default 24h) → BLOCKED.

---

## Stage 5 — D04: Tech Execution

- Spawn D04 Team Lead with the approved master plan and skills mapping.
- Team Lead manages Dev Agents in parallel batches. Do not interfere.
- Trigger Overview Agent every 3 Team Lead iterations (write trigger to `overview/trigger.md`).
- Wait for Team Lead to set `D04.status: DONE` or `PARTIAL`.
- PARTIAL tasks: Team Lead re-queues them. Check again next iteration.
- When all tasks DONE → set `current_stage: D05`.

---

## Stage 6 — D05: Security Audit

- Spawn D05 Security Lead with the implementation path.
- Expected output: `security/findings.md`.
- If findings contain items tagged `REQUIRES_FIX`:
  - Spawn fix cycle: D04 Team Lead (fix only) → D05 re-verify.
  - Increment `D05.fix_cycles`. If fix_cycles ≥ 2 with unresolved REQUIRES_FIX → BLOCKED (security escalation).
- Gate D CLEARED → set `current_stage: D06`.

---

## Stage 7 — D06: Reconciliation

- Spawn D06 Reconciliation Lead with the final implementation and security clearance.
- Expected output: updated `openspec/specs/{slug}-spec.md` (delta-specs merged).
- Constitution Steward processes `shared-proposals/` if any proposals exist.
- Gate E COMPLETE → set `current_stage: DONE`.

---

## Stage 8 — Close

- Write `final-report.md` (see output_artifacts in AGENT.md for required fields).
- Update `pipeline-state.md`: `status: COMPLETE`.
- Report success to user: one paragraph summarizing gates passed, agents used, and any deferred items.

---

## Gate Validator Sub-Agent

Spawn at Gate A. Max iterations: 1. Lives at `departments/orchestrator/gate-validator/AGENT.md`.

Spawn prompt:
```
Read the spec at {spec_path}.
Apply your five consistency rules.
Write verdict to {verdict_path}: first line must be PASS or FAIL.
If FAIL, list each violation as a numbered item below.
```
