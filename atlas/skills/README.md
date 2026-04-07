# Skills — ATLAS Capability Modules

Skills give agents deep, domain-specific expertise without permanently consuming their context windows. A skill is a self-contained file that packages rules, patterns, anti-patterns, and checklists for a particular technology, architecture pattern, or workflow.

**A well-designed skill is 1–3k tokens. If a skill exceeds 5k tokens, it covers too many concerns — split it by sub-concern.**

---

## How Skills Are Loaded (Never Browse This Folder Directly)

Agents do not browse this folder. Skills flow through a structured pipeline:

```
Scout (D01)          reads constitution.md stack + codebase
  └─► scout-report.md   required-skills annotated per module

Skills Mapper (D03)  reads scout-report.md + task list
  └─► skills-mapping.md  maps each task ID → specific skill files

Team Lead (D04)      reads skills-mapping.md at spawn time
  └─► Dev Agent spawn context includes only the relevant skill files
```

An agent whose AGENT.md does not list a skill category in its `skills:` field will never receive a skill from that category, even if the Skills Mapper identifies it as relevant.

---

## Three Design Constraints

1. **Selective loading.** An agent receives only the skills identified as required for its current task. Never the entire folder.
2. **Stack filtering.** `constitution.md` declares the project's technology stack. Skills that do not match the declared stack are excluded from all skill mappings.
3. **Composability.** Skills combine naturally. A task implementing a domain aggregate in C# loads `csharp-domain.skill.md` + `ddd-aggregates.skill.md`. They do not conflict because each addresses a different concern.

---

## Folder Structure

```
skills/
├── README.md                          # This file
├── skill.template.md                  # Copy to create a new skill
│
├── language/                          # Language + layer-specific patterns
│   ├── csharp-domain.skill.md
│   ├── csharp-application.skill.md
│   ├── csharp-infrastructure.skill.md
│   ├── csharp-api.skill.md
│   ├── python-fastapi.skill.md
│   ├── typescript-react.skill.md
│   └── angular-component.skill.md
│
├── architecture/                      # Architecture patterns (stack-agnostic)
│   ├── clean-architecture.skill.md
│   ├── ddd-aggregates.skill.md
│   ├── cqrs-pattern.skill.md
│   └── hexagonal-ports.skill.md
│
├── testing/                           # Testing patterns and strategies
│   ├── unit-testing.skill.md
│   ├── integration-testing.skill.md
│   └── e2e-testing.skill.md
│
├── security/                          # Security rules and patterns
│   ├── owasp-top10.skill.md
│   ├── auth-patterns.skill.md
│   └── input-validation.skill.md
│
├── devops/                            # Infrastructure and CI/CD
│   ├── dockerfile.skill.md
│   ├── github-actions.skill.md
│   └── aws-cdk.skill.md
│
├── documentation/                     # Docs and ADR writing
│   ├── adr-writing.skill.md
│   ├── api-docs.skill.md
│   └── changelog.skill.md
│
├── spec-writing/                      # Spec format and conventions
│   ├── rfc2119-requirements.skill.md
│   ├── given-when-then.skill.md
│   └── delta-spec.skill.md
│
├── code-quality/                      # Review and refactoring
│   ├── code-review.skill.md
│   ├── refactoring-patterns.skill.md
│   └── naming-conventions.skill.md
│
└── custom/                            # Project-specific skills
    └── .gitkeep
```

---

## Skill Categories

| Category | Used By | Stack Filter |
|---|---|---|
| `language/` | Dev Agents, QA Agents | Yes — filtered by `constitution.md` stack |
| `architecture/` | D01 Spec Writers, D02 Architecture Analyst, Dev Agents, Software Architect (D04) | No |
| `testing/` | QA Agents, Dev Agents (unit tests) | No |
| `security/` | D05 Security Agents, D02 Security Pre-scanner, Dep. Vulnerability Scanner | No |
| `devops/` | Dev Agents on infrastructure tasks | No |
| `documentation/` | D01 (ADR creation), D06 (final docs audit) | No |
| `spec-writing/` | D01 Spec Writers, D04 Dev delta-specs, D06 Spec Consolidator | No |
| `code-quality/` | Dev Agents, D05 Architecture Compliance, Software Architect (D04) | No |

---

## Skills Integration Points Across ATLAS

| Dept | Integration Point | How Skills Are Used |
|---|---|---|
| D01 | Scout reconnaissance | Scout reads `constitution.md` stack and annotates `required-skills` per module in `scout-report.md`. Spec Writers may load spec-writing skills. |
| D01 | Domain Expert validation | Domain Expert loads architecture skills (ddd-aggregates, clean-architecture) to validate spec uses correct bounded contexts. |
| D02 | Architecture review | Architecture Analyst loads architecture skills to compare spec against declared patterns. |
| D02 | Security pre-scan | Security Pre-scanner loads security skills (owasp-top10, auth-patterns) for design-time risk assessment. |
| D03 | Skills Mapper agent | Reads task list + Scout's required-skills. Produces `skills-mapping.md`. |
| D04 | Team Lead spawn context | Reads `skills-mapping.md` and includes relevant skill files in each Dev Agent's spawn context. |

---

## How to Create a Custom Skill

1. **Identify the expertise gap.** A skill is warranted when multiple agents need the same domain-specific knowledge (custom framework, proprietary API, internal coding standard not covered by existing skills).
2. **Choose the correct category.** Place in the category that matches its primary concern. If it doesn't fit, create a new directory under `skills/custom/`.
3. **Copy `skill.template.md`.** Fill all required frontmatter fields. Use a descriptive `name` that matches the filename without extension.
4. **Write the content sections.** Minimum: `<context>`, `<rules>`, `<patterns>`, `<anti-patterns>`. Add `<checklist>` if the skill has verifiable outcomes.
5. **Register awareness.** The Scout will identify it during reconnaissance if project files reference the relevant technology. For custom skills, add it manually to the skills catalog in `constitution.md`.

---

## Anti-Patterns

| Anti-Pattern | Why It Fails |
|---|---|
| **Loading all skills** | Consumes context with irrelevant knowledge. The point is selective loading. |
| **Skills as configuration** | Skills contain knowledge (rules, patterns, checklists), not config (feature flags, env vars). Config belongs in `constitution.md`. |
| **Oversized skills** | A 10k-token skill uses 5% of context. Skills must be 1–3k tokens. Split by sub-concern. |
| **Skills that contradict** | Two simultaneously-loaded skills must never give conflicting rules. Resolve conflicts at authoring time. |
| **Agent-fetched skills** | Agents never browse `skills/` themselves. Skills are delivered by the parent at spawn time. Self-loading breaks context isolation. |
| **Skills without version** | If a skill changes without a version increment, agents spawned before and after behave differently. Always increment `version` when modifying rules or patterns. |
| **Skipping Scout skills identification** | If the Scout does not annotate `required-skills` per module, the Skills Mapper has no input, and agents are spawned without skills. The Scout step is not optional. |
