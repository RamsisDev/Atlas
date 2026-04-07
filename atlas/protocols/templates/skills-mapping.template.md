# skills-mapping.md
# Produced by: Skills Mapper (D03)
# Read by: Team Lead (D04) at spawn time for each Dev Agent
# Do not edit manually — this file is generated from scout-report.md + master-plan.md

---

## stack-filter

```yaml
active-stack: {dotnet | python | typescript | ...}
excluded-skills: [{python-fastapi}, {typescript-react}, ...]
```

---

## task-skill-mapping

```
{TASK-ID}: ({brief task description}, layer: {domain|application|infrastructure|api|ui})
  skills:
    - skills/{category}/{skill-name}.skill.md
    - skills/{category}/{skill-name}.skill.md
```

---

## Example (feature: auth-2fa)

```
TASK-01: (UserCredential entity, layer: domain)
  skills:
    - skills/language/csharp-domain.skill.md
    - skills/architecture/ddd-aggregates.skill.md
    - skills/testing/unit-testing.skill.md

TASK-02: (AuthToken aggregate, layer: domain)
  skills:
    - skills/language/csharp-domain.skill.md
    - skills/architecture/ddd-aggregates.skill.md
    - skills/testing/unit-testing.skill.md

TASK-03: (IAuthRepository interface, layer: domain)
  skills:
    - skills/language/csharp-domain.skill.md
    - skills/architecture/clean-architecture.skill.md

TASK-04: (JwtService, layer: application)
  skills:
    - skills/language/csharp-application.skill.md
    - skills/security/auth-patterns.skill.md
    - skills/testing/integration-testing.skill.md
```

---

## Scout Report Excerpt (input to Skills Mapper)

```
# scout-report.md (excerpt)

## modules

### auth/credentials
files-to-read:   [src/Domain/Auth/UserCredential.cs, ...]
files-to-modify: [src/Domain/Auth/UserCredential.cs]
files-to-create: [src/Domain/Auth/AuthToken.cs]
complexity: 3
required-skills: [csharp-domain, ddd-aggregates, unit-testing]

### auth/tokens
files-to-read:   [src/Application/Auth/JwtService.cs, ...]
files-to-modify: [src/Application/Auth/JwtService.cs]
complexity: 2
required-skills: [csharp-application, auth-patterns, integration-testing]
```

---

## Team Lead Spawn Context Assembly (reference)

When Team Lead spawns a Dev Agent for TASK-01, it assembles:

```
1. AGENT.md              (dev-agent identity + protocol)
2. Task definition       (TASK-01 from master-plan.md)
3. Context files         (contract files, spec section, ralph-state if exists)
4. Skills from mapping:
   - skills/language/csharp-domain.skill.md      (~2k tokens)
   - skills/architecture/ddd-aggregates.skill.md  (~1.5k tokens)
   - skills/testing/unit-testing.skill.md         (~1k tokens)

Total skill overhead: ~4.5k tokens out of 200k context window
```
