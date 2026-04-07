# dept-checkpoint.md
# Department Checkpoint Protocol — written by each department at defined intervals.
# D00 reads this file to determine recovery strategy after a failure.
# The rollback-point references a git commit SHA created at checkpoint time.

---

```yaml
department: {D01 | D02 | D03 | D04 | D05 | D06}
feature: {feature-slug}
phase: {EXECUTING | GATE_PENDING | COMPLETE | FAILED}
progress: "{N} of {total} tasks complete"
last-successful-state: {ISO-8601}
```

---

## artifacts-produced

<!-- Files that are COMMITTED to git at the time of this checkpoint. Safe to keep on rollback. -->

```
- {path/to/file.ext} (COMMITTED)
- {path/to/file2.ext} (COMMITTED)
```

## artifacts-in-progress

<!-- Files that exist on disk but are NOT yet committed. May be lost on rollback. -->

```
- {path/to/wip-file.ext} (UNCOMMITTED)
```

---

## rollback

```yaml
rollback-strategy: {FULL | INCREMENTAL | NONE}
rollback-point: "git-sha:{short-sha}"
```

**Strategy definitions:**
- `FULL` — reset to rollback-point, discard all uncommitted artifacts.
- `INCREMENTAL` — keep committed artifacts, discard only uncommitted ones.
- `NONE` — department is complete or failed before any commits; nothing to roll back.

---

## failure-context

<!-- Only populated when phase = FAILED. Describes what went wrong. -->

```yaml
failure-type: {AGENT_CRASH | GATE_REJECTED | SECURITY_CRITICAL | RETRY_EXCEEDED}
failure-detail: "{human-readable description}"
affected-files:
  - {path/to/affected.ext}
```

---

## When to Write This File

| Department | Checkpoint Triggers |
|---|---|
| D00 | After each gate advancement or retry |
| D01 | After Scout completes; after all Spec Writers complete; after Consistency Check |
| D02 | After each sub-agent (Architecture Analyst, Dependency Checker, Security Pre-scanner, Performance Analyst) completes |
| D03 | After Task Planner completes; after Reconciliation Analyst completes; after Skills Mapper completes |
| D04 | After each parallel group completes; every 3 Team Lead iterations; after QA completes |
| D05 | After initial scan completes; after each fix-request cycle resolves |
| D06 | After Docs Auditor completes; after Spec Consolidator completes; after final-report.md is written |
