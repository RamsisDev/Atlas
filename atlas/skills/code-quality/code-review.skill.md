---
name: code-review
description: >
  Rules and standards for conducting effective code reviews following Google's Engineering
  Practices and Trisha Gee's Code Review Best Practices. Covers what to look for in a
  changelist, reviewer behavior, LGTM standards, and the boundary between automated
  checks and human review. Required by the Architecture Compliance agent (D05), Software
  Architect (D04), and QA agents reviewing implementation quality.
applies-to: [software-architect, qa-agent, architecture-compliance, change-auditor]
stack: none
layer: none
version: 1.0
---

# Code Review

<context>
This skill applies when reviewing a code changelist (PR, diff) for correctness,
design, maintainability, and security. Code review is the last human checkpoint before
merge; it MUST focus on what automated tools cannot verify: design decisions, business
logic correctness, and long-term maintainability. Source: Google Engineering Practices
(google.github.io/eng-practices/review) and Trisha Gee's Code Review Best Practices.
</context>

<rules>
## What to Review (Priority Order)
1. **Design**: Does the change make sense within the system architecture? Does it follow the declared patterns (Clean Architecture, DDD, CQRS)?
2. **Correctness**: Does the code do what it claims? Are edge cases handled? Are invariants enforced?
3. **Security**: Does the change introduce any OWASP Top 10 vulnerabilities or break existing security controls?
4. **Tests**: Are the added tests meaningful? Do they verify behavior, not implementation? Are regression cases covered?
5. **Naming**: Are classes, methods, and variables named to reveal intent?
6. **Documentation**: Are public APIs and non-obvious logic commented? Are ADRs updated for significant decisions?
7. **Style**: Formatting and style SHOULD be enforced by linters, not by reviewers.

## Reviewer Behavior
- Reviewers MUST distinguish between blocking issues (bugs, security flaws, design violations) and non-blocking suggestions (style preferences, minor improvements).
- Non-blocking comments MUST be prefixed with "nit:", "suggestion:", or "optional:" so the author can choose.
- Reviewers MUST NOT block a CL on personal style preferences if the existing code style is followed.
- Reviewers SHOULD respond to every CL within one business day; delayed reviews are a team velocity bottleneck.
- Reviewers MUST be constructive: explain why a change is needed, not only what to change.

## LGTM Standard (Google)
- A reviewer MAY approve (LGTM) a CL that is not perfect but makes the overall codebase better.
- A CL that improves the codebase MUST NOT be blocked indefinitely due to minor issues.
- A CL introducing a security vulnerability or architectural regression MUST be blocked regardless of other improvements.

## Automation Boundary
- Formatting, import ordering, and linting MUST be automated (CI check) and MUST NOT appear in review comments.
- Type errors, null safety violations, and test failures MUST be caught by CI; reviewers MUST NOT need to re-check these.
- Security scanning (SAST) MUST be automated; reviewers focus on design-level security decisions not caught by scanners.
- Reviewers MUST NOT manually verify things that automated tests already cover.

## Scope
- A CL MUST do one thing; reviewers MAY request splitting a CL that mixes unrelated concerns.
- Reviewers MUST NOT request unrelated refactoring outside the scope of the CL (scope creep in review).
- Reviewers MUST NOT ask authors to fix pre-existing issues they did not touch, unless the CL makes those issues worse.
</rules>

<patterns>
## Blocking vs. Non-blocking Comment Examples

```
# BLOCKING — security issue, must fix
🚫 This query concatenates user input into SQL — SQL injection vector.
   Use parameterized queries: `db.execute("SELECT ... WHERE id = ?", (user_id,))`

# BLOCKING — design violation
🚫 OrderService is importing from Infrastructure (EFOrderRepository) directly.
   This violates the Dependency Rule — depend on IOrderRepository instead.

# NON-BLOCKING — nit
nit: Consider renaming `data` to `orderSummaryDto` here for clarity.

# SUGGESTION — optional improvement
suggestion: This could be simplified with a LINQ expression, but the current
            version is also clear. No change required.

# QUESTION — seeking understanding, not requesting change
q: I see you chose EF Core's raw SQL here instead of LINQ — was there a
   performance reason? Happy to discuss if helpful.
```

## Review Checklist Template (per CL)

```markdown
## Code Review Checklist

### Design
- [ ] Follows declared architecture (Clean Architecture / DDD / CQRS)
- [ ] No layer boundary violations
- [ ] No unnecessary abstraction (YAGNI)

### Correctness
- [ ] Happy path works as described in the spec
- [ ] Error and edge cases handled
- [ ] No off-by-one or null reference risks

### Security
- [ ] No SQL/command injection vectors
- [ ] No hardcoded secrets or credentials
- [ ] Authorization checked for all resource access

### Tests
- [ ] Tests cover happy path and at least one failure path
- [ ] Test names describe behavior, not implementation
- [ ] No mocks of the class under test

### Naming & Readability
- [ ] Public APIs named to reveal intent
- [ ] Non-obvious logic has a comment explaining *why*, not *what*
```
</patterns>

<anti-patterns>
- **Nit-flood**: 30 "nit:" comments on spacing, variable names, and comma placement — review noise drowns out blocking issues; automate style enforcement.
- **Blocking on personal preference**: "I would have used a switch expression here" when an if-chain is equally clear — imposes opinion without improving quality.
- **Reviewing scope beyond the CL**: asking the author to also fix a pre-existing unrelated bug — scope creep delays merge and discourages refactoring.
- **No distinction between blocking/non-blocking**: author cannot tell which comments must be addressed before merge and which are suggestions.
- **Praising then burying the critical issue**: three paragraphs of praise followed by one sentence noting a security flaw — critical issues should be stated first, clearly.
- **Replicating automated checks**: commenting on missing semicolons or wrong import order when CI already enforces this — reviewer time wasted.
- **Reviewing without understanding the spec**: blocking a CL for missing a behavior that was explicitly descoped in the delta-spec — review the spec, not assumptions.
</anti-patterns>

<checklist>
- [ ] Design reviewed first — architecture violations identified before style issues.
- [ ] All blocking comments explain what AND why.
- [ ] Non-blocking comments prefixed with "nit:", "suggestion:", or "optional:".
- [ ] No comments on formatting or imports already enforced by linter/CI.
- [ ] Security review covers OWASP concerns (injection, auth, access control, logging).
- [ ] Tests reviewed for behavioral coverage, not just coverage percentage.
- [ ] CL does one thing — splitting requested if multiple unrelated concerns mixed.
- [ ] Review completed within one business day of assignment.
</checklist>
