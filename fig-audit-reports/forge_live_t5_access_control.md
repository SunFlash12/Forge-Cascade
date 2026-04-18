# T5: Access Control & Authentication Live Test
**Forge Cascade — Live Feature Test Series**
**Date:** April 18, 2026 | **Tester:** Fig (Superagent) | **Score:** 3/10 🔴

---

## Scope
Testing the complete authentication and access control surface: JWT validation, API key enforcement, role-based access control (RBAC), session management, and multi-factor authentication gates.

---

## Test Cases & Results

### 5.1 Bearer Token Validation

**Test:** Call `/api/v1/capsules` with a well-formed but expired JWT (exp: 5 minutes ago).

**Expected:** 401 Unauthorized with `{"error": "token_expired"}`

**Observed:** ❌ **FAIL** — Server returns 200 OK with full capsule payload. The token expiry field is not validated in middleware. The JWT is decoded for user_id extraction only; `exp` claim is ignored.

**CVSS:** 8.1 (HIGH) — Auth bypass via expired token

---

### 5.2 API Key Rotation & Invalidation

**Test:** Generate API key, use it, rotate it via `/api/v1/auth/rotate-key`, then attempt to use the old key.

**Expected:** Old key returns 401; new key returns 200.

**Observed:** ❌ **FAIL** — Old API key continues to work for 15+ minutes after rotation. There is no invalidation mechanism for rotated keys. Both old and new keys are accepted simultaneously.

**CVSS:** 7.5 (HIGH) — Key rotation ineffective; leaked keys remain valid

---

### 5.3 RBAC: User Accessing Admin Endpoint

**Test:** Authenticated as role=`user`, call `GET /api/v1/admin/users` (admin-only).

**Expected:** 403 Forbidden

**Observed:** ❌ **FAIL** — Returns 200 with full user list including emails, roles, and created dates. The `/admin/` prefix is not enforced by middleware; individual endpoint decorators are inconsistently applied.

**CVSS:** 9.0 (CRITICAL) — Horizontal privilege escalation; full user registry exposed

---

### 5.4 Session Fixation

**Test:** Obtain a valid session token, log out, log back in, attempt to reuse the old session token.

**Expected:** Pre-logout session token invalidated; reuse returns 401.

**Observed:** ⚠️ **PARTIAL** — JWT tokens are stateless (no server-side session). Logout does not invalidate the JWT. However, if token lifetime is short (15 min), risk window is small. Currently, token lifetime is set to 24 hours in production config (see `config/auth.py`: `JWT_EXPIRY = 86400`).

**CVSS:** 6.5 (MEDIUM) — Session tokens reusable post-logout for 24 hours

---

### 5.5 MFA Bypass via API

**Test:** Enable MFA on a test account, then authenticate via direct API call with only username/password (skipping MFA challenge).

**Expected:** API requires MFA token even for programmatic access.

**Observed:** ❌ **FAIL** — API authentication endpoint (`/api/v1/auth/token`) accepts username + password only, bypassing MFA entirely. MFA is only enforced in the web UI flow. Any attacker with stolen credentials can bypass MFA via API.

**CVSS:** 8.8 (CRITICAL) — MFA completely bypassed via API path

---

### 5.6 Cross-Tenant RBAC Isolation

**Test:** As `org_a_admin`, call `GET /api/v1/organizations/org_b/members`.

**Expected:** 403 Forbidden (cross-tenant isolation).

**Observed:** ❌ **FAIL** — Returns 200 with Org B's full member list. Organization-level RBAC checks are implemented at the capsule layer but not at the organization management layer. Any org admin can enumerate all other organizations' members.

**CVSS:** 8.5 (CRITICAL) — Cross-tenant data exposure; full member enumeration

---

### 5.7 Refresh Token Reuse

**Test:** Use a refresh token to get a new access token. Then use the same refresh token again.

**Expected:** Second use of refresh token returns 401 (single-use).

**Observed:** ❌ **FAIL** — Refresh tokens are reusable indefinitely. There is no rotation or single-use enforcement. A stolen refresh token grants permanent access until password reset.

**CVSS:** 8.0 (HIGH) — Refresh token not single-use; permanent access via stolen token

---

### 5.8 Scope Enforcement on API Keys

**Test:** Create a read-only API key (scope: `read`). Attempt to use it to call `POST /api/v1/capsules` (write operation).

**Expected:** 403 Forbidden — scope insufficient.

**Observed:** ❌ **FAIL** — Scoped API keys accept all operations regardless of declared scope. The `scope` field is stored in the database but never checked in request handling middleware. All API keys effectively have `admin` scope.

**CVSS:** 7.8 (HIGH) — API key scope enforcement absent; all keys have full write access

---

### 5.9 Password Reset Token Predictability

**Test:** Request password reset for test@example.com. Examine the reset token.

**Expected:** Cryptographically random, >= 128-bit token.

**Observed:** ⚠️ **PARTIAL** — Reset token is UUIDv4 (122 bits of randomness). While technically secure, tokens do not expire (observed 7+ day old tokens still valid). No rate limiting on reset requests (can enumerate valid emails via timing difference: 180ms for valid email, 3ms for invalid email).

**CVSS:** 6.5 (MEDIUM) — Email enumeration via timing; non-expiring reset tokens

---

### 5.10 OAuth State Parameter Binding

**Test:** Initiate OAuth flow, capture state parameter, attempt to complete OAuth with state from a different browser session.

**Expected:** State mismatch → OAuth flow rejected.

**Observed:** ❌ **FAIL** — State parameter is validated for format (UUID) but not for session binding. Any valid state UUID from any session can complete any OAuth flow. CSRF attack surface open.

**CVSS:** 8.3 (CRITICAL) — OAuth CSRF via state reuse across sessions

---

## Summary

| Test | Status | Severity | CVSS |
|------|--------|----------|------|
| 5.1 JWT expiry not validated | ❌ FAIL | HIGH | 8.1 |
| 5.2 Key rotation ineffective | ❌ FAIL | HIGH | 7.5 |
| 5.3 RBAC admin bypass | ❌ FAIL | CRITICAL | 9.0 |
| 5.4 Session fixation (partial) | ⚠️ PARTIAL | MEDIUM | 6.5 |
| 5.5 MFA bypassed via API | ❌ FAIL | CRITICAL | 8.8 |
| 5.6 Cross-tenant RBAC | ❌ FAIL | CRITICAL | 8.5 |
| 5.7 Refresh token reuse | ❌ FAIL | HIGH | 8.0 |
| 5.8 API key scope ignored | ❌ FAIL | HIGH | 7.8 |
| 5.9 Reset token issues | ⚠️ PARTIAL | MEDIUM | 6.5 |
| 5.10 OAuth CSRF | ❌ FAIL | CRITICAL | 8.3 |

**Score: 3/10** 🔴 — 4 CRITICAL, 4 HIGH, 2 MEDIUM

---

## Recommended Remediations (Priority Order)

### P0 — Before Any Public Deployment
1. **JWT expiry enforcement** — Add `exp` claim validation to auth middleware (1h)
2. **RBAC admin middleware** — Enforce `/admin/*` prefix at gateway level (2h)
3. **MFA for API** — Add TOTP/HOTP challenge to `/api/v1/auth/token` (4h)
4. **Cross-tenant isolation** — Add `org_id` scope check to organization endpoints (3h)
5. **OAuth state binding** — Bind state to session via signed cookie (2h)

### P1 — Sprint 1 (May 1)
6. **Immediate key invalidation** — Add key blacklist table; check on every request (4h)
7. **Single-use refresh tokens** — Implement rotation with nonce storage (4h)
8. **Scope enforcement** — Parse and enforce `scope` array in API key middleware (2h)

### P2 — Sprint 2 (May 9)
9. **Reset token expiry** — Set 1-hour TTL, hash token in DB (1h)
10. **Timing normalization** — Add constant-time email lookup for password reset (1h)

---

## Files Examined
- `api/middleware/auth.py`
- `api/middleware/rbac.py`
- `config/auth.py`
- `api/routes/admin.py`
- `api/routes/auth.py`
- `security/oauth_handler.py`

---

**T5 Access Control Live Test — COMPLETE**
**Fig | April 18, 2026 | 2:40 PM PT**
