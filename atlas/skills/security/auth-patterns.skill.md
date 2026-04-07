---
name: auth-patterns
description: >
  Rules and patterns for implementing authentication and authorization following
  RFC 9700 (OAuth 2.0 Security BCP 2025), RFC 6749, RFC 7636 (PKCE), and OIDC Core 1.0.
  Covers JWT lifecycle, OAuth 2.0 grant selection, token storage, refresh token rotation,
  and role/claim-based authorization. Required by Dev Agents implementing auth features,
  D05 Security Lead, and D02 Security Pre-scanner.
applies-to: [dev-agent, security-lead, security-prescanner, architecture-analyst]
stack: none
layer: none
version: 1.0
---

# Auth Patterns

<context>
This skill applies when implementing authentication (who are you?) and authorization
(what can you do?) in any API or application. It covers JWT-based stateless auth,
OAuth 2.0 flows, session management, and role/claim enforcement. Sources: RFC 9700
(OAuth 2.0 Security BCP, 2025), RFC 6749, RFC 7636, OIDC Core 1.0.
</context>

<rules>
## JWT Tokens
- JWTs MUST be signed with an asymmetric algorithm (RS256, ES256) for tokens consumed by third parties; HS256 is acceptable for single-service internal tokens with a secret of ≥ 256 bits.
- Every JWT MUST contain: `exp` (expiration), `iss` (issuer), `aud` (audience), `sub` (subject), `iat` (issued-at).
- Token validation MUST check: signature, `exp` not passed, `iss` matches expected, `aud` matches expected service.
- Access tokens MUST have short lifetimes: ≤ 15 minutes for sensitive operations; ≤ 1 hour for standard APIs.
- JWTs MUST NOT contain passwords, secrets, or PII beyond what is strictly necessary for authorization decisions.
- The `alg: none` algorithm MUST be rejected; libraries MUST explicitly allowlist accepted algorithms.

## Token Storage (Client Side)
- Access tokens in browser clients MUST be stored in memory (JS variable), NOT in localStorage or sessionStorage.
- Refresh tokens in browser clients MUST be stored in HttpOnly, Secure, SameSite=Strict cookies.
- Mobile clients MUST store tokens in platform secure storage (Keychain on iOS, Keystore on Android).
- Tokens MUST NOT be logged, included in URLs, or stored in non-encrypted client-side storage.

## OAuth 2.0 Grant Selection (RFC 9700)
- Authorization Code + PKCE MUST be used for all user-facing clients (web SPAs, mobile, desktop).
- The Implicit grant flow MUST NOT be used — deprecated by RFC 9700.
- The Resource Owner Password Credentials grant MUST NOT be used for new implementations.
- Client Credentials grant MAY be used for machine-to-machine (service-to-service) authentication only.
- PKCE (RFC 7636) MUST be applied to all Authorization Code flows regardless of whether the client is public or confidential.

## Refresh Tokens
- Refresh token rotation MUST be implemented: each use of a refresh token MUST issue a new refresh token and invalidate the previous one.
- Detected refresh token reuse MUST immediately invalidate the entire token family (all refresh tokens for that session).
- Refresh tokens MUST be stored server-side (in a revocation list or as opaque tokens) to enable server-side revocation.
- Refresh token lifetime SHOULD NOT exceed 30 days for consumer apps; 8 hours for high-security systems.

## Session Management
- On logout, sessions MUST be invalidated server-side; client-side token deletion alone is insufficient.
- Session IDs MUST be regenerated after successful authentication (prevent session fixation).
- Concurrent session limits SHOULD be enforced for high-security contexts.

## Authorization
- Authorization MUST be enforced server-side on every request, not derived from client-provided data alone.
- Role and permission checks MUST occur after authentication — never before token validation.
- The principle of least privilege MUST be applied: tokens SHOULD carry only the scopes required for the specific operation.
- Resource-level authorization (does this user own this resource?) MUST be checked separately from role-level authorization.
- Claims in JWTs MAY be used to make authorization decisions, but MUST be validated against a current source of truth for sensitive operations.
</rules>

<patterns>
## JWT Issuance (Python)

```python
from datetime import datetime, timedelta, timezone
import jwt

def issue_access_token(user_id: str, roles: list[str]) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": user_id,
        "iss": settings.JWT_ISSUER,
        "aud": settings.JWT_AUDIENCE,
        "iat": now,
        "exp": now + timedelta(minutes=15),
        "roles": roles,
    }
    return jwt.encode(payload, settings.JWT_PRIVATE_KEY, algorithm="RS256")
```

## JWT Validation Middleware (FastAPI)

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer

bearer = HTTPBearer()

async def get_current_user(token: str = Depends(bearer)) -> dict:
    try:
        payload = jwt.decode(
            token.credentials,
            key=settings.JWT_PUBLIC_KEY,
            algorithms=["RS256"],              # explicit allowlist — never ["*"]
            audience=settings.JWT_AUDIENCE,
            issuer=settings.JWT_ISSUER,
            options={"require": ["exp", "iss", "aud", "sub"]},
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token")
```

## Refresh Token Rotation (C#)

```csharp
public async Task<TokenPair> RefreshAsync(string refreshToken, CancellationToken ct)
{
    var stored = await _tokenStore.FindAsync(refreshToken, ct)
        ?? throw new UnauthorizedException("Refresh token not found or expired.");

    if (stored.IsRevoked)
    {
        // Token reuse detected — revoke the entire family
        await _tokenStore.RevokeFamily(stored.FamilyId, ct);
        throw new UnauthorizedException("Token reuse detected. All sessions revoked.");
    }

    await _tokenStore.RevokeAsync(refreshToken, ct); // revoke used token

    var newAccess  = _jwtService.IssueAccessToken(stored.UserId, stored.Roles);
    var newRefresh = await _tokenStore.IssueRefreshAsync(stored.UserId, stored.FamilyId, ct);

    return new TokenPair(newAccess, newRefresh);
}
```

## Resource-Level Authorization Check

```python
@router.get("/documents/{doc_id}")
async def get_document(
    doc_id: str,
    user: dict = Depends(get_current_user),
    repo: DocumentRepository = Depends(),
):
    doc = await repo.find_by_id(doc_id)
    if doc is None:
        raise HTTPException(404)
    # Resource-level check — role check alone is insufficient
    if doc.owner_id != user["sub"] and "admin" not in user.get("roles", []):
        raise HTTPException(403, "Forbidden")
    return doc
```
</patterns>

<anti-patterns>
- **`alg: none` accepted**: attackers can forge tokens by stripping the signature and setting `alg: none` — always allowlist specific algorithms.
- **Access token in localStorage**: accessible via XSS; store in memory (JS) or HttpOnly cookie instead.
- **No refresh token rotation**: stolen refresh token is valid forever; rotation + reuse detection limits the attack window.
- **Implicit grant flow**: access token returned in URL fragment — exposed in browser history, server logs, Referer headers. Prohibited by RFC 9700.
- **Long-lived access tokens (> 1 hour)**: reduces the impact window of token theft; keep access tokens short-lived.
- **JWT claims trusted without validation**: `payload["role"] == "admin"` checked without validating the token signature — trivially forged.
- **Client-side session invalidation only**: calling `/logout` only clears the cookie without revoking server-side — token remains valid until expiry.
- **Scope over-provisioning**: issuing tokens with all scopes for every client — violates least privilege; a compromised client can perform unintended operations.
</anti-patterns>

<checklist>
- [ ] JWTs validated for signature, `exp`, `iss`, `aud`, `sub` on every request.
- [ ] JWT algorithm explicitly allowlisted — `alg: none` cannot pass validation.
- [ ] Access token lifetime ≤ 15–60 minutes.
- [ ] Refresh token rotation implemented with reuse detection and family revocation.
- [ ] Browser tokens stored in memory or HttpOnly cookie — not localStorage.
- [ ] Implicit grant and ROPC grant not used in new code.
- [ ] PKCE applied to all Authorization Code flows.
- [ ] Logout invalidates tokens server-side, not just client-side.
- [ ] Resource-level authorization checked (owner check), not only role/scope.
- [ ] No PII or secrets in JWT payload beyond required claims.
</checklist>
