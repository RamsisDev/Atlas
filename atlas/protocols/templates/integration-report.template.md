# Integration Report Template
# Location: departments/tech/work/{feature}/integration-report.md
# Written by Integration Agent (D04-03) after all Dev groups complete, before QA.
# A FAIL status blocks QA — Team Lead must route fixes and re-validate.

---

# integration-report.md | feature: {feature-name}

reviewer: integration-agent | timestamp: {ISO8601}
status: PASS | FAIL

## contract-drift
{For each contract file, compare declared public API against actual implementation.}
- {ClassName}.contract.md → MATCH | DRIFT: {description of drift}
  {If DRIFT: list missing methods, changed signatures, different return types.}

## cross-boundary-calls
{For each task with dependencies (deps field), verify the consuming code uses the
contract's declared API — not an assumed or outdated API.}
- {ConsumerClass} → {ProviderClass}.{MethodName}: MATCH | MISMATCH
  Parameter: {paramName} ({type}), return: {returnType}
  {If MISMATCH: what the code actually calls vs. what the contract declares.}

## reconciliation-compliance
{Verify merge order and integration points match the reconciliation-strategy in master-plan.md.}
- Merge order {layer1} → {layer2} → {layer3}: FOLLOWED | VIOLATED
- Team Lead review points executed: {list of checkpoints}

## missing-contracts
{Tasks that modified a public API but did not write a contract file.}
- {TASK-ID} ({devId}): exposes {ClassName} — no contract file found
- None

## issues (if FAIL)
{Specific remediation instructions per responsible dev.}
- dev-{N}: {TASK-ID} — {specific fix instruction}
  Drift: {ClassName}.{methodName} declared as returning {Type} but implementation returns {Type2}.
  Fix: update implementation to match contract, OR write {ClassName}.v2.contract.md
  and append MINOR to pending-triage.md.

---

## Validation Checklist

| Check | Condition for PASS |
|---|---|
| Contract drift | Every public method in every contract exists in the implementation with matching signature |
| Cross-boundary calls | All consuming code uses only methods declared in the relevant contract |
| Reconciliation compliance | Merge order matches reconciliation-strategy in master-plan.md |
| Missing contracts | Every task that exposed a public API has a contract file |
