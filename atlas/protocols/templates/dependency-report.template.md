# dependency-report.md | feature: {feature}

scanner: dependency-vuln-scanner | timestamp: {ISO-8601}
status: CLEAN | ISSUES_FOUND

## new-dependencies
- {package@version} (added by {dev-id}, {TASK-ID})
  CVEs: {none | CVE-YYYY-NNNN (severity: HIGH|MEDIUM|LOW, patched in version)}
  maintained: yes | no (last publish: {YYYY-MM})
  verdict: CLEAN | UPDATE_REQUIRED | STALE

## updated-dependencies
- {package@version} (updated from {old-version} by {dev-id}, {TASK-ID})
  CVEs: {none | CVE-YYYY-NNNN}
  maintained: yes | no
  verdict: CLEAN | UPDATE_REQUIRED | STALE

## stale-dependencies
- {package@version} — last publish: {YYYY-MM} ({N} years ago)
  verdict: STALE

## recommendation
{Actionable recommendation for each UPDATE_REQUIRED or STALE package.}
