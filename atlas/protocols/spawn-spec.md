# Spawn Primitives Specification

ATLAS uses three spawn primitives throughout all protocols. This file defines exactly what each one means. Without this definition, "spawn Group A tasks as parallel batch" is ambiguous and non-reproducible.

---

## Primitives

### spawn(agent, context)

Invokes a single agent with a defined context.

```
agent:    path to AGENT.md
context:  list of files the agent receives as input
returns:  agent exit status + path to output artifacts
```

**Constraints:**
- Agent receives ONLY its AGENT.md, task definition, and listed context files.
- Agent does NOT inherit parent's conversation history.
- Agent writes output to its designated work directory.
- Agent follows its own Ralph protocol independently.

---

### spawn_batch(agents[], contexts[])

Spawns N agents in parallel. All must reach terminal state before the caller continues.

```
agents:   array of paths to AGENT.md files
contexts: array of context file lists (one per agent)
returns:  array of { status, output_artifacts } (one per agent)
```

**Constraints:**
- All agents run concurrently with independent context windows.
- No agent can read another agent's in-progress work.
- Caller blocks until ALL agents reach terminal state.
- Terminal states: COMPLETE, PARTIAL, FAILED.
- Caller processes ALL results before deciding whether to extend PARTIAL agents or create overflow tasks.

---

### spawn_sequential(agents[], contexts[])

Spawns agents one at a time. Output artifacts of agent N are added to the context of agent N+1.

```
agents:   array of paths to AGENT.md files (execution order)
contexts: array of base context file lists (one per agent)
returns:  array of { status, output_artifacts } (one per agent)

Execution:
  for i in 0..N-1:
    result[i] = spawn(agents[i], contexts[i] + result[i-1].output_artifacts)
    if result[i].status == FAILED: abort remaining, return results so far
```

Use when each agent's work depends on the previous agent's output. Example: Reconciliation Analyst must read Task Planner's output before producing the execution plan.

---

## Context Isolation Principle

The spawned agent receives only what is explicitly passed to it. This prevents three failure categories:

| Failure | Cause | Prevention |
|---|---|---|
| **Context pollution** | Dev Agent inherits 50k tokens of Team Lead orchestration context, leaving less room for actual work | Explicit context list only |
| **Implicit coupling** | Agent B reads Agent A's conversation history; changes to A silently affect B | No history inheritance |
| **Non-reproducibility** | Agents depend on conversation state; same inputs produce different results depending on conversation path | Clean context window per spawn |

> Every spawned agent starts with a clean context window. Its only inputs are AGENT.md, the task definition, explicitly listed context files, and relevant skill files. Nothing else.

---

## Spawn Context Assembly

When a parent spawns a child, it assembles the child's context from four sources in this order:

1. **AGENT.md** — the child's identity, model, tools, disallowed tools, max turns, skills list, and behavioral protocol. Always the first content in the prompt.
2. **Task definition** — the specific assignment from `master-plan.md`, including task ID, description, file-scope, dependencies, and parallel group.
3. **Context files** — explicitly listed files the agent needs to read. For a Dev Agent: contract files for dependencies, the relevant spec section, and any existing ralph-state file.
4. **Skill files** — loaded on demand from `skills-mapping.md`. Only the skills identified as required for this specific task. A Dev Agent working on domain layer receives `csharp-domain.skill.md` and `ddd-aggregates.skill.md`, not the entire skills folder.

**Rules:**
- The parent is responsible for assembling the context correctly.
- The child never fetches its own context — it works exclusively with what it receives at spawn time.
- If a required file is missing from the context, the child reports it via `pending-triage.md` as a MAJOR inconsistency rather than attempting to locate it independently.

---

## Claude Code Reference Implementation

In Claude Code, the three primitives map directly to the Agent tool:

| Primitive | Claude Code Mapping |
|---|---|
| `spawn()` | Single Agent tool call. Prompt = AGENT.md contents + task definition. `allowedTools` and `disallowedTools` set from AGENT.md fields. `maxTurns` set from AGENT.md `max_iterations`. |
| `spawn_batch()` | Multiple Agent tool calls issued in the same `function_calls` block. Claude Code executes them concurrently. Each call is independent with its own prompt, tools, and maxTurns. Caller waits for all to complete before processing results. |
| `spawn_sequential()` | Sequential Agent tool calls where each call includes the previous call's output artifacts in its prompt. The caller issues one call, waits for completion, reads the output, and includes it in the next call's prompt. |

**Example — Team Lead spawning parallel Group A (two Dev Agents):**

```
# Team Lead issues two Agent tool calls in one function_calls block:

Agent call 1:
  prompt: [AGENT.md for dev-agent] + [TASK-01 definition] + [contract files]
  allowedTools: [Read, Write, Bash, Glob, Grep]
  disallowedTools: []
  maxTurns: 7    # score(5) + 2

Agent call 2:
  prompt: [AGENT.md for dev-agent] + [TASK-03 definition] + [contract files]
  allowedTools: [Read, Write, Bash, Glob, Grep]
  disallowedTools: []
  maxTurns: 6    # score(4) + 2

# Both run concurrently. Team Lead processes results when both complete.
```

---

## API Reference Implementation

For custom orchestrators using the Anthropic API directly:

| Primitive | API Mapping |
|---|---|
| `spawn()` | New API conversation. System prompt constructed from AGENT.md. First user message contains task definition and context file contents. Tools configured per AGENT.md specification. |
| `spawn_batch()` | `Promise.all()` of N independent API conversations. Each conversation is fully isolated. Orchestrator collects all results before proceeding. |
| `spawn_sequential()` | Chain of API conversations where each conversation's output artifacts are serialized and included in the next conversation's first user message as additional context. |

---

## Spawn Lifecycle

Every spawned agent follows the same lifecycle regardless of which primitive created it:

```
1. CREATED     Parent assembles context, issues spawn call
2. RUNNING     Agent executes within its Ralph protocol
3. CHECKPOINT  Agent writes ralph-state at 80% budget (optional mid-run)
4. TERMINAL    Agent reaches one of:
               - COMPLETE:         All work finished, output artifacts written
               - PARTIAL:          Budget exhausted, some work remains
               - EARLY_EXIT_WARNING: 80% budget, signaling it may not finish
               - FAILED:           Unrecoverable error, no state written
5. COLLECTED   Parent reads result, decides next action
```

**For `spawn_batch()`:** all agents must reach terminal state before the parent processes any results. This prevents decisions based on incomplete information.

**For `spawn_sequential()`:** a FAILED status aborts the remaining chain immediately and returns all results collected so far.
