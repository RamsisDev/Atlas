# master-plan.md | feature: {feature-name}

## dev-agents-required: {N}
## qa-agents-required: {N}
## ralph-iterations-estimated: score + 2 per dev (see Fractal Budget)
## skills-mapping: departments/pm/work/{feature}/skills-mapping.md

---

## assignments

{dev-agent-id}:
  - id: TASK-{NN}
    description: {description} (score:{total}, layer:{layer})
    parallel_group: {A|B|C|...}
    deps: [{TASK-IDs} | []]
    file-scope: [{file1}, {file2}]
    required-skills: [{skill-category-1}, {skill-category-2}]

  - id: TASK-{NN}
    description: {description} (score:{total}, layer:{layer})
    parallel_group: {A|B|C|...}
    deps: [{TASK-IDs} | []]
    file-scope: [{file1}]
    required-skills: [{skill-category-1}]

---

## parallel-execution-plan

Group {A} (spawn as batch): {TASK-IDs}
  → {rationale: no file overlap / safe parallel}

Group {B} (after Group {A}): {TASK-IDs}
  → depends on {TASK-IDs}

Group {C} (after Group {B}): {TASK-IDs}
  → depends on {TASK-IDs}

Team Lead review points: after Group {A}, after Group {last}

---

## reconciliation-strategy

Merge order: {layer1} → {layer2} → {layer3}
File conflicts anticipated: {none | list conflicts}
Team Lead review points: {same as parallel-execution-plan}

---

## gate-c-timeout-hours: 24

## blocker (if any)
{NONE | description of blocker that requires human or D01 intervention}
