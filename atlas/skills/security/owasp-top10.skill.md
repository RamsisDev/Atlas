---
name: owasp-top10
description: >
  Security rules and prevention patterns derived from the OWASP Top 10 2021.
  Covers the ten most critical web application security risks with concrete prevention
  rules and anti-patterns for each category. Load this skill for any agent reviewing
  code that handles user input, authentication, authorization, data access, or
  dependency management. Required by D05 Security agents and D02 Security Pre-scanner.
applies-to: [dev-agent, security-lead, change-auditor, architecture-compliance, security-prescanner]
stack: none
layer: none
version: 1.0
---

# OWASP Top 10 (2021)

<context>
This skill applies during security reviews of any code that: accepts external input,
performs authentication or authorization, queries databases, renders output to clients,
or manages dependencies. Each section maps to one OWASP Top 10 category and provides
prevention rules usable as gate criteria in D05 reviews.
</context>

<rules>
## A01 — Broken Access Control
- Every API endpoint MUST verify that the authenticated user is authorized to access the requested resource.
- Authorization checks MUST be enforced server-side; client-side checks are supplementary only.
- Indirect object references (IDs in URLs) MUST be validated against the requesting user's ownership or role.
- Directory listing MUST be disabled on all web servers serving application files.
- CORS policies MUST allowlist specific trusted origins; wildcard (`*`) MUST NOT be used on authenticated endpoints.

## A02 — Cryptographic Failures
- Sensitive data (passwords, tokens, PII, payment data) MUST be encrypted at rest and in transit.
- Passwords MUST be hashed with an adaptive algorithm: bcrypt (cost ≥ 12), Argon2id, or scrypt.
- MD5, SHA-1, and plain SHA-256 MUST NOT be used for password hashing.
- TLS MUST be enforced for all connections; HTTP-only endpoints MUST NOT exist in production.
- Secrets, API keys, and connection strings MUST NOT be hardcoded in source code or committed to version control.

## A03 — Injection
- All database queries MUST use parameterized queries or prepared statements; string concatenation with user input is PROHIBITED.
- ORM query builders MUST NOT be bypassed with raw SQL that interpolates user input.
- Shell commands MUST NOT be constructed from user input; if shell execution is unavoidable, use allowlisted arguments only.
- Output rendered to HTML MUST be context-escaped (HTML, attribute, JavaScript, CSS contexts treated separately).
- LDAP, XML, and OS command injection prevention MUST follow the same parameterization principle.

## A04 — Insecure Design
- Security requirements MUST be identified in the design phase (D01/D02), not only during D05.
- Threat modeling SHOULD be performed for features handling sensitive data or privileged operations.
- Rate limiting MUST be designed into authentication endpoints at the architecture level.
- The principle of least privilege MUST be applied to all system accounts, service roles, and user roles.

## A05 — Security Misconfiguration
- Default credentials MUST be changed before any deployment (dev, staging, prod).
- Error responses MUST NOT expose stack traces, internal paths, or database schema details to clients.
- HTTP security headers MUST be set: `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`, `Strict-Transport-Security`.
- Unnecessary features, endpoints, and services MUST be disabled.
- Dependency versions MUST be pinned; auto-update without review MUST NOT be enabled in production.

## A06 — Vulnerable and Outdated Components
- All direct and transitive dependencies MUST be scanned for known CVEs before each deployment.
- Dependencies with CRITICAL or HIGH severity CVEs MUST be updated or mitigated before Gate E approval.
- A software bill of materials (SBOM) SHOULD be generated and stored per release.
- Unmaintained dependencies (no releases in 2+ years with open CVEs) MUST be replaced.

## A07 — Identification and Authentication Failures
- Passwords MUST meet minimum complexity requirements enforced server-side.
- Authentication endpoints MUST implement account lockout or exponential backoff after repeated failures.
- Session tokens MUST be invalidated server-side on logout; client-side deletion alone is insufficient.
- Multi-factor authentication SHOULD be supported for privileged accounts.
- JWT tokens MUST be validated for signature, expiry (`exp`), issuer (`iss`), and audience (`aud`) on every request.

## A08 — Software and Data Integrity Failures
- CI/CD pipelines MUST verify artifact integrity (checksums or signatures) before deployment.
- Deserialization of untrusted data MUST NOT instantiate arbitrary classes; use allowlisted types only.
- Auto-update mechanisms MUST verify signatures before applying updates.

## A09 — Security Logging and Monitoring Failures
- All authentication events (success and failure) MUST be logged with timestamp, user identity, and IP.
- All authorization failures MUST be logged.
- Logs MUST NOT contain passwords, tokens, or full PII.
- Log entries MUST be tamper-resistant; logs MUST NOT be writable by the application process.

## A10 — Server-Side Request Forgery (SSRF)
- Any user-controlled URL or hostname MUST be validated against an allowlist of permitted destinations.
- Internal network ranges (169.254.x.x, 10.x.x.x, 172.16–31.x.x, 192.168.x.x) MUST be blocked for outbound requests triggered by user input.
- DNS rebinding protections MUST be applied when following HTTP redirects from user-supplied URLs.
</rules>

<patterns>
## Parameterized Query (Python)

```python
# CORRECT — parameterized, user input never touches the SQL string
async def get_user(db: AsyncSession, user_id: int) -> User | None:
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()

# WRONG — SQL injection vector
query = f"SELECT * FROM users WHERE id = {user_id}"  # NEVER do this
```

## Password Hashing (Python)

```python
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")

def hash_password(plain: str) -> str:
    return pwd_context.hash(plain)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

## JWT Validation (Python)

```python
import jwt

def decode_token(token: str) -> dict:
    return jwt.decode(
        token,
        key=settings.JWT_SECRET,
        algorithms=["HS256"],
        options={"require": ["exp", "iss", "aud"]},
        audience=settings.JWT_AUDIENCE,
        issuer=settings.JWT_ISSUER,
    )
    # jwt.ExpiredSignatureError, jwt.InvalidTokenError raised automatically
```

## Security Headers (middleware)

```python
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["Strict-Transport-Security"] = "max-age=63072000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        return response
```
</patterns>

<anti-patterns>
- **String-concatenated SQL**: `f"SELECT * FROM users WHERE name = '{name}'"` — SQL injection vector; always parameterize.
- **MD5/SHA-1 for passwords**: both are reversible with rainbow tables in seconds; use Argon2id or bcrypt.
- **Hardcoded secrets**: `API_KEY = "sk-..."` in source code — rotatable secret exposed in version history forever.
- **Wildcard CORS on authenticated endpoints**: `Access-Control-Allow-Origin: *` with `Authorization` headers enabled — allows any origin to read responses.
- **Stack traces in HTTP responses**: `500 Internal Server Error: NullReferenceException at MyApp.Service.Process(...)` — reveals internal structure to attackers.
- **Client-side-only authorization**: checking roles in JavaScript without server-side enforcement — trivially bypassed via browser devtools.
- **Unvalidated redirects**: `redirect(request.args.get("next"))` — open redirect vector for phishing.
- **Logging full tokens or passwords**: `logger.info(f"Login attempt with password={password}")` — secrets appear in log files.
</anti-patterns>

<checklist>
- [ ] All SQL/ORM queries use parameterized inputs — no string interpolation of user data.
- [ ] Passwords hashed with Argon2id, bcrypt (cost ≥ 12), or scrypt.
- [ ] No secrets, API keys, or credentials appear in source files.
- [ ] JWT tokens validated for signature, `exp`, `iss`, and `aud` on every request.
- [ ] HTTP security headers set: CSP, X-Frame-Options, X-Content-Type-Options, HSTS.
- [ ] All authorization checks are server-side (not client-side only).
- [ ] User-controlled URLs validated against an allowlist before making outbound requests (SSRF).
- [ ] Dependencies scanned for CVEs; no CRITICAL/HIGH unmitigated findings.
- [ ] Auth events (success + failure) logged with timestamp, identity, IP — no passwords in logs.
</checklist>
