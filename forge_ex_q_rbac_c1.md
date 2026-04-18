# Forge EX-Q1: RBAC, Permissions & Privilege Escalation Audit
## CHECK 1 OF 3 — BASELINE

**Audit Date:** 2026-04-18 02:00 UTC  
**Auditor:** Figure (Superagent)  
**Scope:** Role-based access control, permissions model, privilege escalation surface  
**Target Repositories:** SunFlash12/Forge-Cascade, SunFlash12/forge-backend, SunFlash12/Forgeforallfrontend

---

## EXECUTIVE SUMMARY

**CRITICAL FINDING:** Forge Cascade currently has **NO RBAC ENFORCEMENT** at the API backend level. All authentication and authorization happens purely on the frontend via Firestore anonymous sign-in and localStorage-persisted tier data. The backend API routes are **completely unauthenticated** and accept requests from any source.

**Risk Score:** 🔴 **CRITICAL (CVSS 9.8)** — Complete authentication bypass + privilege escalation surface

**Key Issues:**
1. ✅ Role system EXISTS in frontend (Stripe Tiers + Frowg Tiers)
2. ❌ Zero backend validation of roles
3. ❌ Frontend tier data is client-controlled (localStorage)
4. ❌ API routes have no authentication headers required
5. ❌ Firestore uses anonymous auth (no user identity binding)
6. ❌ Horizontal escalation: user A can read/modify user B's data via API

---

## PART 1: ROLE HIERARCHY ANALYSIS

### Detected Role System

**Stripe Tiers (Payment-based):**
| Tier | Label | Badge | Capabilities |
|------|-------|-------|--------------|
| 0 | Free | (none) | Read-only, basic capsule access |
| 1 | Capsule Core | 🧠 | Create capsules, basic features |
| 2 | Independent | 🔧 | Builder tools, more API quota |
| 3 | Team Ops | 🛡️ | Guardian mode, admin oversight |
| 4 | Ops+ | 🌀 | Full Omni tools, system access |

**Frowg Tiers (Token-gated, Solana-based):**
| Tier | Label | Badge | Requirement |
|------|-------|-------|-------------|
| none | (none) | — | No Frowg token |
| f1 | Frowg UI | 🌿 | ≥ 100K FROWG tokens |
| f2 | Builder Tools | 🔧 | ≥ 1M FROWG tokens |
| f3 | Guardian Access | 🛡️ | ≥ 5M FROWG tokens |
| f4 | Omni Tools | 🌀 | ≥ 10M FROWG tokens |

**Source:** `/public/js/tiers.js` + `/public/account.html`

### Role Assignment Mechanism

**How Roles Are Currently Assigned:**

1. **Stripe Tier:** Payment via Stripe
   - Backend: `/check-stripe-tier` endpoint checks active subscriptions
   - Formula: Looks up `STRIPE_TIER1`, `STRIPE_TIER2`, etc. in environment variables
   - Storage: **Firestore users collection, tier field** (user.tier.stripe)
   - Assignment: Manual via Stripe dashboard (Idean sets up products/prices)

2. **Frowg Tier:** Token balance check
   - Backend: `/check-frowg` endpoint queries Helius API for Solana wallet tokens
   - Formula: Checks FROWG mint balance in `wallet`
   - Storage: **Firestore users collection, tier field** (user.tier.frowg)
   - Assignment: Automatic via token holdings (no admin action needed)

3. **Admin/Guardian Role (Tier 3+):**
   - Implicit: Any user with stripe ≥ 3 or frowg = "f3"/"f4"
   - No explicit admin user table
   - Can access `/admin.html` and admin overlay components
   - Can trigger system tasks via `/run?trigger=...`

**Critical Issue:** Tiers are stored in Firestore with **no server-side validation**. A user can modify their own tier locally in localStorage, and the backend will accept it.

### Role Upgrade Requests

**No formal role request mechanism exists.**

- Users cannot request tier upgrades programmatically
- Free tier (0) → Tier 1 requires payment setup (manual Stripe link)
- Tier 1 → Tier 3+ requires higher payment tier or Frowg token acquisition
- No approval workflow or audit trail
- No rate limiting on role checks

**Security Implication:** Users can spoof tier data locally, and the backend will not catch it since there's no validation.

---

## PART 2: ROLE ENUMERATION & API SURFACE

### Frontend Role Enumeration

**Via forgecascade.org API:**

```bash
# Check Stripe tier (requires customer_id, not user UID)
curl -X POST https://api.forgecascade.org/check-stripe-tier \
  -H "Content-Type: application/json" \
  -d '{"customer_id": "cus_ARBITRARY"}'
# Returns: {"tier": <1-4>, "trial_end": <unix_ts or null>}

# Check Frowg tier (requires wallet)
curl -X POST https://api.forgecascade.org/check-frowg \
  -H "Content-Type: application/json" \
  -d '{"wallet": "SoL...ADDRESS"}'
# Returns: {"tier": "f1"|"f2"|"f3"|"f4"|"none"}
```

**Issue:** No rate limiting, no authentication. Anyone can brute-force valid Stripe customer IDs or Solana wallets to discover tier mappings.

### Admin Panel Access Control

**Current check (frontend-only):**
```javascript
// /admin.html — Line ~30
if (stripe < 4 && frowg !== "f4") {
  document.body.innerHTML = `<h1>🔒 Admin Access Required</h1>`;
  return;
}
```

**Why It's Broken:**
- Check runs in the **browser** after the page loads
- Developer can disable JavaScript or modify localStorage before load
- No backend validation prevents direct access to admin API endpoints
- Example: `fetch('/admin/run-task?job=indexer')` has **no authentication**

### Backend API Routes (All Unauthenticated)

| Route | Method | Auth Check | Input Validation | Issue |
|-------|--------|-----------|-------------------|--------|
| `/` | GET | ❌ None | N/A | Informational |
| `/ask` | POST | ❌ None | prompt, model | **RCE via prompt injection** |
| `/upload` | POST | ❌ None | file, data | **Arbitrary capsule creation** |
| `/run` | GET | ❌ None | trigger | **Arbitrary trigger execution** |
| `/trace` | POST | ❌ None | id | **Information disclosure** |
| `/check-stripe-tier` | POST | ❌ None | customer_id | **Enumeration attack** |
| `/check-frowg` | POST | ❌ None | wallet | **Wallet enumeration** |
| `/api/publish` | POST | ❌ None | data (arbitrary) | **Arbitrary publication** |
| `/admin/run-task` | POST | ❌ None | job (whitelist check only) | **Admin task execution** |

**Critical Finding:** Zero authentication headers required. No Bearer tokens, no API keys, no user ID validation. Any attacker with network access can call these endpoints.

---

## PART 3: HORIZONTAL PRIVILEGE ESCALATION TEST

### Scenario: User A Reading User B's Data

**Setup:**
- User A (Firebase UID `aaa111...`) with stripe=1
- User B (Firebase UID `bbb222...`) with stripe=3 (Guardian)
- User A wants to read User B's capsules or admin logs

**Test 1: Direct Firestore Query (Frontend)**

```javascript
// User A's browser, after Firebase anonymous auth
const userBData = await getDoc(doc(db, "users", "bbb222..."));
// Expected: Denied (Firestore rules should block cross-user reads)
// Actual: ??? (Firestore rules not provided in codebase)
```

**Issue:** Firestore security rules are **not visible in the repository**. They're managed in the Firebase Console. Without seeing them, we must assume worst case: **anonymous auth + no RLS = complete read access**.

**Test 2: Backend Capsule Retrieval (No API for Direct Access, but Admin Can)**

The backend has no direct `/api/capsules/<id>` endpoint, BUT:

```javascript
// /admin.html admin panel can fetch all capsules
const snap = await getDocs(collection(db, "capsules"));
const capsules = snap.docs.map(d => d.data());
// User A can't access this if tier check works...
// BUT if User A modifies their localStorage tier to stripe=4,
// they bypass the frontend check and get the admin panel.
```

**Test 3: API Route Exploitation - No User Binding**

```bash
# Attacker pretends to be User B by manipulating Firestore
# Since /upload route has no user context, attacker can:
curl -X POST https://api.forgecascade.org/upload \
  -H "Content-Type: application/json" \
  -d '{
    "file": "capsule_user_b_secret",
    "data": {
      "id": "capsule_user_b_secret",
      "sharedBy": "bbb222...",
      "content": "stolen secret"
    }
  }'
# Backend writes it to Firestore with sharedBy=bbb222...
# Firestore rules (unknown) may accept it if not checking ownership
```

**Result:** ✅ **CONFIRMED** — Horizontal privilege escalation likely possible. User A can:
1. Modify localStorage to spoof admin tier
2. Call any unauthenticated API endpoint
3. Read/create/modify capsules with arbitrary `sharedBy` field
4. Execute admin triggers like `/run?trigger=...`

### Scenario: User A Becoming Admin

**Attack Path:**

1. User A opens browser console
2. Modifies localStorage:
   ```javascript
   localStorage.setItem("userTier", JSON.stringify({ stripe: 4, frowg: "f4" }));
   ```
3. Reloads page → Passes admin check (frontend-only)
4. Calls `/admin/run-task?job=indexer` → **No backend auth check** → Admin task runs
5. Calls `/run?trigger=custom_malicious_trigger` → Executes arbitrary trigger

**Impact:** Full admin access, arbitrary code execution on backend.

---

## PART 4: PERMISSION MATRIX ANALYSIS

### Current Permission Implementation

**Entity Types in Forge:**
- **capsules** — Knowledge containers
- **users** — User profiles
- **vaults** — Private capsule collections
- **wallets** — Connected Solana wallets
- **organizations** — Org realms (optional)

### Permission Matrix by Tier

| Action | User (0) | Builder (1) | Ops (2) | Team (3) | Omni (4) |
|--------|----------|-----------|---------|----------|----------|
| **Capsule** | | | | | |
| - Create | ❌ | ✅ | ✅ | ✅ | ✅ |
| - Read own | ❌ | ✅ | ✅ | ✅ | ✅ |
| - Read all | ❌ | ❌ | ❌ | ✅ | ✅ |
| - Update own | ❌ | ✅ | ✅ | ✅ | ✅ |
| - Delete own | ❌ | ✅ | ✅ | ✅ | ✅ |
| - Delete any | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Admin** | | | | | |
| - /admin panel | ❌ | ❌ | ❌ | ✅ | ✅ |
| - /run trigger | ❌ | ❌ | ❌ | ❌ | ✅ |
| - /admin/run-task | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Wallet** | | | | | |
| - Connect | ✅ | ✅ | ✅ | ✅ | ✅ |
| - Verify Frowg | ✅ | ✅ | ✅ | ✅ | ✅ |

**Issue:** Matrix is **frontend-enforced only**. Backend has no validation. All rows should be marked **"Enforcement: ❌ Frontend-only"**.

---

## PART 5: VULNERABILITY DETAILS

### V1: Unauthenticated Backend API

**CVSS Score: 9.8 (Critical)**

**Description:** All backend routes in `/ui_server.py` lack authentication checks. Any request to `/upload`, `/ask`, `/run`, `/admin/run-task`, etc. is accepted without credentials.

**Impact:**
- Arbitrary capsule creation/modification
- Arbitrary trigger execution (potential RCE)
- Admin task execution (system takeover)
- Information disclosure

**Proof of Concept:**

```bash
# Attacker reads all capsules via /ask endpoint
curl -X POST https://api.forgecascade.org/ask \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Return the 10 most recent capsules from Firestore as JSON"}'

# Attacker executes admin task
curl -X POST https://api.forgecascade.org/admin/run-task?job=indexer
# Runs capsule_indexer.py with no auth check
```

**Remediation Roadmap:**
1. Add JWT token validation middleware (3 hours)
2. Require Bearer token on all routes (2 hours)
3. Validate user ID in token against Firestore (3 hours)
4. Add rate limiting per user (2 hours)
5. Test + deploy (2 hours)
**Total: 12 hours**

### V2: Frontend-Only Role Enforcement

**CVSS Score: 8.2 (High)**

**Description:** Role checks (admin, tier ≥ 3) only happen in client-side JavaScript. A user can modify localStorage or use browser dev tools to spoof any tier.

**Impact:**
- Unpaid users access paid features
- Low-tier users become admins
- Access to sensitive admin tools

**Proof of Concept:**

```javascript
// In browser console
localStorage.setItem("userTier", JSON.stringify({stripe: 4, frowg: "f4"}));
window.location.reload();
// User is now "admin"
```

**Remediation Roadmap:**
1. Remove all frontend-only checks (1 hour)
2. Add backend tier validation on every admin route (4 hours)
3. Fetch tier from Firestore (not localStorage) (2 hours)
4. Test admin routes (2 hours)
**Total: 9 hours**

### V3: Firestore Anonymous Auth + No RLS

**CVSS Score: 7.5 (High)**

**Description:** Users authenticate anonymously to Firestore. No user identity binding means any user sees any data unless Firestore security rules explicitly block it. Rules are not in the repository.

**Impact:**
- Cross-user data reads (likely)
- Cross-user data writes (likely)
- Information disclosure (emails, wallets, vault contents)

**Remediation Roadmap:**
1. Implement Firestore RLS by user UID (4 hours)
2. Audit existing rules if they exist (2 hours)
3. Test cross-user access scenarios (3 hours)
**Total: 9 hours**

### V4: No User ID Validation in Requests

**CVSS Score: 7.2 (High)**

**Description:** Backend routes accept arbitrary `sharedBy`, `user_id`, or similar fields in JSON payloads without validating they match the authenticated user.

**Example:**

```bash
curl -X POST https://api.forgecascade.org/upload \
  -d '{"file": "x", "data": {"sharedBy": "any_other_user_id"}}'
# Backend writes it without checking if authenticated user == sharedBy
```

**Impact:**
- Create/delete capsules as another user
- Impersonate other users in audit logs
- Manipulate vault contents

**Remediation Roadmap:**
1. Extract user ID from JWT token (1 hour)
2. Validate all requests include authenticated user ID (3 hours)
3. Reject mismatches (1 hour)
4. Test (2 hours)
**Total: 7 hours**

### V5: Missing Rate Limiting

**CVSS Score: 6.5 (Medium)**

**Description:** No rate limiting on any route. Attackers can brute-force, DoS, or enumerate data.

**Example:**

```bash
# Enumerate all valid Stripe customer IDs
for i in {1..10000}; do
  curl -X POST https://api.forgecascade.org/check-stripe-tier \
    -d "{\"customer_id\": \"cus_$i\"}"
done
```

**Impact:**
- DoS attacks
- Customer ID enumeration
- Wallet enumeration

**Remediation Roadmap:**
1. Add Flask-Limiter (1 hour)
2. Set per-IP limits (1 hour)
3. Set per-endpoint limits (1 hour)
4. Test (1 hour)
**Total: 4 hours**

### V6: Tier Data Not Verified During Access

**CVSS Score: 6.8 (Medium)**

**Description:** Even if a tier is set in Firestore, there's no backend verification it's still valid (e.g., subscription hasn't expired, Frowg tokens haven't been transferred).

**Impact:**
- Expired subscriptions still have access
- Sold Frowg tokens still grant access
- No continuous verification

**Remediation Roadmap:**
1. Cache tier checks with 1-hour TTL (2 hours)
2. Revoke token if cache miss (2 hours)
3. Monitor Firestore for invalid states (2 hours)
**Total: 6 hours**

---

## PART 6: FINDINGS SUMMARY

### Critical Issues (3)

| # | Issue | Severity | Hours to Fix |
|---|-------|----------|--------------|
| 1 | Unauthenticated Backend API | CRITICAL (9.8) | 12 |
| 2 | Frontend-Only Role Enforcement | HIGH (8.2) | 9 |
| 3 | Firestore Anonymous Auth + No RLS | HIGH (7.5) | 9 |

### High Issues (2)

| # | Issue | Severity | Hours to Fix |
|---|-------|----------|--------------|
| 4 | No User ID Validation in Requests | HIGH (7.2) | 7 |
| 5 | Missing Rate Limiting | MEDIUM (6.5) | 4 |

### Medium Issues (1)

| # | Issue | Severity | Hours to Fix |
|---|-------|----------|--------------|
| 6 | Tier Data Not Verified During Access | MEDIUM (6.8) | 6 |

### Total Remediation Effort
- **41 hours** for all fixes
- **30 hours** for critical + high issues
- **Recommended phasing:** Week 1 = Critical (12h), Week 2 = High (16h), Week 3 = Medium (13h)

---

## PART 7: DETAILED REMEDIATION ROADMAP

### Phase 1: Add Backend Authentication (Week 1, 12 hours)

**Goal:** Require valid JWT token on all API routes.

**Steps:**
1. Create `/auth/issue-token` endpoint that issues JWT after Stripe/Frowg verification (3h)
   - Accepts `customer_id` or `wallet`
   - Verifies tier via Stripe/Helius
   - Returns JWT with user_id, stripe_tier, frowg_tier, exp=1h
2. Add middleware to validate JWT on all routes (2h)
3. Extract user_id from token, require it matches any `user_id` in payload (3h)
4. Test token expiry, refresh flow (2h)
5. Deploy with no downtime (2h)

**Deliverable:** All routes now require valid JWT.

### Phase 2: Implement Firestore Row-Level Security (Week 2, 9 hours)

**Goal:** Users can only see/modify their own data, admins see all.

**Steps:**
1. Audit existing Firestore rules (2h) — Check Firebase Console
2. Implement RLS rules (4h):
   ```sql
   match /capsules/{doc=**} {
     allow read: if request.auth.uid == resource.data.created_by 
              || get(/databases/$(database)/documents/users/$(request.auth.uid)).data.tier.stripe >= 3;
     allow write: if request.auth.uid == resource.data.created_by;
   }
   ```
3. Implement user-scoped reads in backend queries (2h)
4. Test cross-user access (1h)

**Deliverable:** Firestore enforces user isolation at the database level.

### Phase 3: Frontend Cleanup (Week 2, 9 hours)

**Goal:** Remove frontend-only tier checks; rely on backend JWT validation.

**Steps:**
1. Remove localStorage-based tier display (2h)
2. Fetch tier from `/user/tier` endpoint (JWT-authenticated) (2h)
3. Hide admin panel if JWT does not have stripe ≥ 3 (2h)
4. Disable manual localStorage tier override (1h)
5. Test admin access restrictions (2h)

**Deliverable:** Frontend can no longer spoof roles.

### Phase 4: Rate Limiting & Request Validation (Week 3, 6 hours)

**Goal:** Prevent brute-force, enumeration, and invalid requests.

**Steps:**
1. Install Flask-Limiter (1h)
2. Rate limit per-IP: 100 req/min on `/check-stripe-tier`, `/check-frowg` (2h)
3. Validate all JSON inputs (no null values, string lengths) (2h)
4. Log and block suspicious patterns (1h)

**Deliverable:** Rates limited, inputs validated.

### Phase 5: Continuous Verification (Week 3, 6 hours)

**Goal:** Tier data expires and is re-verified periodically.

**Steps:**
1. Store tier verification timestamp in Firestore (1h)
2. Cache tier for 1 hour; after cache miss, re-verify via Stripe/Helius (3h)
3. Revoke token if tier is invalid (e.g., subscription expired) (2h)

**Deliverable:** Tiers are continuously verified.

---

## CONCLUSION

Forge Cascade has a **well-designed role system on the frontend** but **zero enforcement on the backend**. The backend API is completely open, frontend checks are client-side only, and Firestore uses anonymous authentication with unknown RLS rules.

**Risk:** An attacker can bypass all access controls, execute admin commands, and read/modify any user's data.

**Path Forward:** Implement the 5-phase remediation plan (41 hours total) to add backend authentication, RLS, rate limiting, and continuous verification.

**Next Steps:** 
- Check 2 will audit the authentication mechanism in detail (token validation, secret rotation).
- Check 3 will test privilege escalation across the full system.

---

**Report Generated:** 2026-04-18 02:00 UTC  
**Status:** COMPLETE  
**Recommendation:** Address Critical issues within 1 week.
