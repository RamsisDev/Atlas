# verdict.md

## status: APPROVED | REJECTED
## date: {ISO-8601}

## architecture-analyst findings
- [OK] Clean Architecture respected
- [WARN] {component} crosses boundary — document in ADR
- [VIOLATION] {specific violation}

## dependency-checker findings
- [BREAK] {service} assumes {constraint} — affected by this change
- [OK] Remaining services not dependent

## security-pre-scanner findings
- [RISK] {finding}: {what is missing in spec}
- [WARN] {finding}: {recommendation}
- [OK] {component}: {control specified}

## performance-analyst findings
- [OK] {operation}: {why it is safe}
- [WARN] {operation}: {concern and recommendation}
- [RISK] {operation}: {bottleneck} — {what is missing}

## conditions-for-approved (if REJECTED)
1. {Specific action required — reference spec section}
2. {Specific action required — reference spec section}
3. {Specific action required — reference spec section}
