# ATLAS - Autonomous Task Loop with Agent Specialization [![Spec Version](https://img.shields.io/badge/ATLAS-v1.0-blue)](./docs/ATLAS_v1_0_Unified.md) [![License](https://img.shields.io/badge/license-CC%20BY--NC%204.0-lightgrey)](./LICENSE) [![Status](https://img.shields.io/badge/status-early%20development-olive)](./ROADMAP.md)

> A complete, production-ready specification and implementation library for multi-department autonomous software development pipelines powered by Claude.


---

## What is ATLAS?

ATLAS transforms a natural-language feature request into implemented, tested, security-audited, and reconciled code - with no manual hand-off between stages.

It is built on six established patterns unified into a single coherent pipeline:

| Pattern | Role in ATLAS |
|---|---|
| **BMAD** | Departmental structure with specialized persona-agents per role |
| **Fractal Decomposition** | Tasks scored across 6 dimensions, decomposed recursively when complexity exceeds threshold |
| **Ralph Loop** | Every agent serializes state to disk at 80% context; parent re-spawns with fresh window |
| **OpenSpec** | Delta-based specs (ADDED / MODIFIED / REMOVED) accumulate as the living source of truth |
| **Scout Pattern** | Reconnaissance sub-agent reads the codebase and calculates workload before any spec writing |
| **Overview Agent** | Pull-based triage agent triggered by Team Lead at defined checkpoints |

ATLAS is not a replacement for lighter methods. A five-line bug fix does not need eight departments. ATLAS is designed for projects where multiple agents must coordinate across multiple architectural layers, where security auditing and reconciliation are mandatory, and where the specification must remain the authoritative source of truth throughout the entire lifecycle.

---

## Architecture overview

```
User
 └─► D00 Pipeline Orchestrator
      ├─► D01 Analysis & Spec Design      [Gate A: consistency check]
      ├─► D02 Viability Review            [Gate B: APPROVED / REJECTED]
      ├─► D03 Project Management          [Gate C: human approval]
      ├─► D04 Tech Execution ◄──► Overview Agent
      │    ├─ Team Lead
      │    ├─ Dev Agents × N (parallel batches)
      │    ├─ QA Agent (parallel groups)
      │    ├─ Software Architect Agent
      │    └─ Integration Agent
      ├─► D05 Security Audit              [Gate D: Team Lead sign-off]
      └─► D06 Reconciliation             [Gate E: security clearance]
           └─► Constitution Steward (batches shared-proposals/)
```

**33 total agents** · **7 departments** · **2 transversal agents** · **5 quality gates** · **8 skill categories**

---

## Core principles

**1. Spec-first execution.** No agent executes without an approved specification. No change to code exists without a corresponding delta-spec.

**2. No context loss.** At 80% context usage or iteration limit, every agent writes its complete state to disk and stops cleanly. The parent re-spawns it with a fresh window. State lives on the filesystem, not in memory.

**3. Pull, not push.** No agent runs as a background process. The Overview Agent is triggered by the Team Lead every N iterations. All coordination is pull-based.

**4. Skills-driven specialization.** Skills are reusable capability modules loaded on demand per task. A Dev Agent working on domain layer loads `csharp-domain.skill.md` and `ddd-aggregates.skill.md`. A QA Agent testing APIs loads `integration-testing.skill.md` and `owasp-top10.skill.md`.

**5. Deterministic coordination by design.** File-scope overlap is validated at Gate C before execution begins. If D03 guarantees no two tasks in the same parallel group touch the same file, runtime locks are unnecessary - and eliminated entirely.

---

## Repository structure

```
atlas/
├── docs/
│   ├── ATLAS_v1_0_Unified.md          # Complete system specification
│   ├── SDD_Methods_Guide.md           # 8 SDD methods comparison
│   ├── Fractal_BMAD_v2.md             # Previous version (reference)
│   └── Gap_Resolution_Proposals.md   # 15 gap analyses (integrated into v1.0)
│
├── departments/
│   ├── orchestrator/          # D00 — Pipeline Orchestrator
│   │   ├── AGENT.md
│   │   └── pipeline-state.md  # Template
│   ├── analysis/              # D01 — Analysis & Spec Design
│   ├── viability/             # D02 — Viability Review
│   ├── pm/                    # D03 — Project Management
│   ├── tech/                  # D04 — Tech Execution
│   ├── security/              # D05 — Security Audit
│   ├── reconciliation/        # D06 — Reconciliation
│   └── overview/              # Overview Agent (transversal)
│
├── skills/
│   ├── languages/             # csharp/, python/, typescript/, ...
│   ├── architecture/          # ddd/, clean-arch/, cqrs/, ...
│   ├── testing/               # unit/, integration/, e2e/, ...
│   ├── security/              # owasp/, threat-modeling/, ...
│   ├── devops/                # docker/, terraform/, github-actions/, ...
│   ├── documentation/         # adr/, openapi/, ...
│   ├── spec-writing/          # rfc2119/, given-when-then/, ...
│   └── custom/                # Your project-specific skills
│
├── openspec/
│   ├── constitution.md        # Project principles (never edited directly)
│   ├── constitution-proposals/
│   ├── shared-proposals/
│   ├── architecture/
│   ├── specs/                 # Approved specs - single source of truth
│   └── changes/               # Active deltas + archive
│
├── examples/
│   ├── auth-2fa/              # Full worked example: 2FA feature end-to-end
│   └── api-endpoint/          # Simpler example: single REST endpoint
│
└── model-capabilities.md      # Context thresholds per Claude model
```

---

## Implementation phases

The system is built in four phases. Each phase unlocks the next.

### Phase 1 — Make it executable
*Nothing runs without these.*

- [x] Spawn primitives: `spawn()`, `spawn_batch()`, `spawn_sequential()` — Claude Code and API implementations
- [x] D00 Pipeline Orchestrator + Gate Validator agent
- [x] `delta-spec.md` standard format (ADDED / MODIFIED / REMOVED)
- [x] Skills architecture: folder structure + `.skill.md` format
- [x] Universal Ralph Protocol: `{agent}-state.md` template and 8-step contract

### Phase 2 — Make it safe
*Recovery, coordination, no race conditions.*

- [x] File-scope overlap check at Gate C (replaces the entire `locks/` directory)
- [x] Department checkpoint protocol (`dept-checkpoint.md` + D00 recovery strategies)
- [x] Interface Contract protocol (`{Entity}.contract.md` in `contracts/`)
- [x] Software Architect Agent (D04, read-only, max 2 iterations)
- [x] Integration Agent (D04, contract verification + integration testing, max 3 iterations)

### Phase 3 — Make it performant
*Parallel QA, multi-feature isolation, better triage.*

- [x] Parallel QA groups (QA Root: 6 iters · QA sub-agents: 2 iters each)
- [x] Overview Agent read-only code access (+Read, Glob, Grep tools)
- [x] Multi-feature isolation via git branches + `shared-proposals/`
- [x] Escalation Router agent + Dependency Vulnerability Scanner (D05)
- [x] Constitution Steward (batches all `shared-proposals/` in a single controlled pass)

### Phase 4 — Polish
*Metrics, scoring refinement, minor agents.*

- [ ] Unified two-phase scoring with calibration feedback loop
- [ ] Standard process metrics in `final-report.md` + Metrics Compiler agent
- [ ] Model capabilities matrix + configurable Gate C timeout
- [ ] Consistency Checker: 5 defined validation rules (RFC 2119 + Given/When/Then)
- [ ] Domain Expert (D01), Performance Analyst (D02), Skills Mapper (D03)

---

## Agent roster

| Agent | Department | Max iterations | Status |
|---|---|---|---|
| Pipeline Orchestrator | D00 | 10 | Phase 1 |
| Gate Validator | D00 | 1 | Phase 1 |
| General Analyst | D01 | 2 | Phase 1 |
| Scout | D01 | 5 | Phase 1 |
| Spec Writer × N | D01 | 3 | Phase 1 |
| Consistency Checker | D01 | 1 | Phase 4 |
| Domain Expert | D01 | 2 | Phase 4 |
| Spec Verifier | D02 | 1 | Phase 1 |
| Architecture Analyst | D02 | 1 | Phase 1 |
| Dependency Checker | D02 | 1 | Phase 1 |
| Security Pre-scanner | D02 | 1 | Phase 1 |
| Performance Analyst | D02 | 1 | Phase 4 |
| Project Manager | D03 | 2 | Phase 1 |
| Task Planner | D03 | 2 | Phase 1 |
| Reconciliation Analyst | D03 | 2 | Phase 1 |
| Skills Mapper | D03 | 1 | Phase 4 |
| Team Lead | D04 | 4 | Phase 1 |
| Dev Agent × N | D04 | score + 2 | Phase 1 |
| QA Root Agent | D04 | 6 | Phase 3 |
| QA Test Sub-agent | D04 | 2 | Phase 3 |
| Software Architect Agent | D04 | 2 | Phase 2 |
| Integration Agent | D04 | 3 | Phase 2 |
| Security Lead | D05 | 8 | Phase 1 |
| Change Auditor | D05 | 3 | Phase 1 |
| Prompt Injection Scanner | D05 | 2 | Phase 1 |
| Architecture Compliance | D05 | 2 | Phase 1 |
| Dep. Vulnerability Scanner | D05 | 2 | Phase 3 |
| Reconciliation Lead | D06 | 5 | Phase 1 |
| Docs Auditor | D06 | 2 | Phase 1 |
| Spec Consolidator | D06 | 3 | Phase 1 |
| Metrics Compiler | D06 | 1 | Phase 4 |
| Overview Agent | Transversal | 5 | Phase 1 |
| Constitution Steward | Transversal | 3 | Phase 3 |

---

## Fractal complexity scoring

Every task is scored across six dimensions before execution. Tasks exceeding the threshold decompose recursively.

| Score | Classification | Action |
|---|---|---|
| ≤ 8 | **ATOMIC** | Agent executes directly in Ralph Loop |
| 9–14 | **MEDIUM** | Decompose into 2–3 sub-agents (each must score ≤ 8) |
| 15–19 | **HIGH** | Two-level tree + human gate before executing |
| ≥ 20 | **EXCESSIVE** | Spec is not ready — return to D01 |

Dev Agent iteration budget: `max_iterations = task_score + 2`

---

## Which SDD method to use

ATLAS is one of eight spec-driven development methods covered in this library. Choose the right level of ceremony for your work.

| Your situation | Recommended method |
|---|---|
| Solo dev, ship fast | **GSD** — minimum overhead, maximum velocity |
| Small team, new features | **Spec-Kit** — structured without enterprise overhead |
| Existing codebase, incremental | **OpenSpec** — brownfield-first, delta specs prevent drift |
| Enterprise / regulated industry | **BMAD** — full audit trail, PRD, architecture docs |
| Autonomous overnight runs | **Ralph Loop** — fresh context each iteration |
| AWS ecosystem, IDE-native | **Kiro** — built for this exact profile |
| Complex task graph | **TaskMaster AI** — dependency tracking, provider agnostic |
| Full autonomous pipeline | **ATLAS** — this repository |

---

## Quick start

> Phase 1 implementation is in progress. The following will be the entry point once the execution layer is complete.

```bash
# 1. Clone the repository
git clone https://github.com/your-org/atlas-agent-library
cd atlas-agent-library

# 2. Initialize your project
cp openspec/constitution.md.template openspec/constitution.md
# Edit constitution.md with your project principles, tech stack, and standards

# 3. Create your first feature request
echo "Add JWT authentication with refresh tokens" > feature-request.txt

# 4. Launch the pipeline via D00
# (Claude Code — requires Claude Code with Agent tool enabled)
claude "Read departments/orchestrator/AGENT.md and start the ATLAS pipeline for the feature described in feature-request.txt"
```

---

## Documentation

| Document | Description |
|---|---|
| [SDD Methods Guide](./docs/SDD_Methods_Guide.md) | 8 methods compared: GSD, Ralph Loop, OpenSpec, TaskMaster, Spec-Kit, Kiro, BMAD, Tessl |
| [Optimal Prompt Formats](./docs/Optimal_Prompt_Formats.md) | Evidence-based guide for Claude agent prompts |

---

## Prompt format conventions

All agents in this library follow the conventions documented in the [Optimal Prompt Formats Guide](./docs/Optimal_Prompt_Formats.md). Key decisions:

- **XML** for section boundaries in system prompts (`<instructions>`, `<ralph_protocol>`, `<protocol>`)
- **Markdown** for structure within sections
- **Direct imperative prose** for behavioral rules
- **JSON Schema with `strict: true`** and minimum 3–4 sentences per tool description
- **`<examples>` with nested `<example>` tags** for few-shot patterns

---

## Contributing

This library is under active development. Contributions are welcome across all phases.

**High-priority contributions for Phase 1:**
- AGENT.md files for each department's root agent
- `spawn-spec.md` with Claude Code reference implementation
- Skill files for common stacks (C# + Clean Architecture, Python + FastAPI, TypeScript + NestJS)
- Worked examples in `examples/`

Please read [CONTRIBUTING.md](./CONTRIBUTING.md) before submitting a PR. All agents must include: `prompt.md`, `guide.md`, and at least one example with input and expected output.

---

## License

Creative Commons Attribution-NonCommercial 4.0 International - see [LICENSE](./LICENSE) for details.

---

*Claude Agent Library · April 2026*