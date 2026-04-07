---
name: {skill-name}
description: >
  {3+ sentences describing: what the skill covers, when to use it,
  and what agents benefit from loading it. Be specific enough that
  the Skills Mapper can match it to a task without reading the body.}
applies-to: [dev-agent, qa-agent]
stack: {dotnet | python | typescript | none}
layer: {domain | application | infrastructure | api | ui | Testing | none }
version: 1.0
---

# {Skill Title}

<context>
{When this skill applies. Helps the agent determine if the skill is
relevant to its current task. Be explicit about the conditions.

Example: "This skill applies when implementing domain entities, value objects,
aggregates, and domain events in a C# .NET project following DDD."}
</context>

<rules>
{Mandatory constraints using RFC 2119 keywords. These are non-negotiable.
Each rule is one line. Use MUST, SHALL, SHOULD, MAY.}

- Entities MUST {constraint}
- Value objects MUST {constraint}
- {Component} SHALL NOT {constraint}
- {Component} SHOULD {constraint}
</rules>

<patterns>
{Code examples showing the correct way to implement the skill's domain.
Organize by sub-pattern with Markdown headers.}

## {Pattern Name}

```{language}
// Example showing the correct implementation
```

## {Pattern Name 2}

```{language}
// Example showing the correct implementation
```
</patterns>

<anti-patterns>
{Common mistakes agents must avoid. Each is a concise statement of what NOT to do.
One line per anti-pattern.}

- {Anti-pattern}: {why it fails}
- {Anti-pattern}: {why it fails}
- {Anti-pattern}: {why it fails}
</anti-patterns>

<checklist>
{Optional. Verification checklist the agent (or QA) can use to confirm
the skill was applied correctly. Include only verifiable items.}

- [ ] {Verifiable condition 1}
- [ ] {Verifiable condition 2}
- [ ] {Verifiable condition 3}
</checklist>
