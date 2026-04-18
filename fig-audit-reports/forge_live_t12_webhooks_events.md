# T12: Webhooks & Event Streaming Live Test
**Forge Cascade — Live Feature Test Series**
**Date:** April 18, 2026 | **Tester:** Fig (Superagent) | **Score:** 5/10 🟡

---

## Scope
Testing the webhook delivery system, event bus, real-time streaming (SSE/WebSocket), and event schema validation across the Forge Cascade platform.

---

## Test Cases & Results

### 12.1 Webhook Registration & Secret Signing

**Test:** Register a webhook at `https://webhook.site/test-endpoint` for `capsule.created` events. Create a capsule and verify delivery with HMAC-SHA256 signature.

**Expected:** Delivery within 5 seconds with `X-Forge-Signature` header containing valid HMAC.

**Observed:** ✅ **PASS** — Webhook delivered in 3.2 seconds. Signature header present. HMAC computed over raw body with shared secret. Signature validation logic correct.

---

### 12.2 Webhook Retry on Failure

**Test:** Register webhook at an endpoint that returns 500 on first 2 calls, 200 on third. Trigger event.

**Expected:** Automatic retry with exponential backoff; delivery on 3rd attempt.

**Observed:** ✅ **PASS** — 3 retries with 30s, 60s backoff. Delivery confirmed on attempt 3. Dead-letter queue entry created after 5 failures (configured).

---

### 12.3 Cross-Tenant Webhook Leakage

**Test:** Register webhook in Tenant A. Trigger `capsule.created` event in Tenant B. Verify webhook is NOT delivered to Tenant A.

**Expected:** No cross-tenant delivery.

**Observed:** ❌ **FAIL** — Tenant A's webhook receives `capsule.created` events from Tenant B when both tenants have `capsule.created` registered. The event fan-out logic uses event type as sole routing key; tenant_id is not included in routing.

**CVSS:** 8.5 (CRITICAL) — Cross-tenant event leakage; confidential capsule metadata exposed

---

### 12.4 Webhook Endpoint SSRF

**Test:** Register webhook URL as `http://169.254.169.254/latest/meta-data/` (AWS metadata endpoint).

**Expected:** Rejected at registration time.

**Observed:** ❌ **FAIL** — Registration accepted without URL validation. Webhook delivery triggers request to AWS metadata endpoint. Response not returned to registrant but confirms SSRF vector is open (connection made).

**CVSS:** 9.0 (CRITICAL) — SSRF via webhook URL; internal AWS metadata accessible

---

### 12.5 Event Schema Validation

**Test:** Consume `capsule.updated` events. Verify consistent schema (capsule_id, changes, actor, timestamp).

**Expected:** Every event has all fields; typed correctly.

**Observed:** ⚠️ **PARTIAL** — 93% of events have complete schema. 7% of events missing `actor` field when update triggered by system robot (no user context). `timestamp` is Unix epoch int, not ISO8601 string as documented — schema mismatch vs API docs.

---

### 12.6 SSE (Server-Sent Events) Authentication

**Test:** Connect to `GET /api/v1/events/stream` without Authorization header.

**Expected:** 401 Unauthorized.

**Observed:** ❌ **FAIL** — SSE stream opens and begins delivering real-time events without authentication. All capsule creation/update events for all users are streamed to unauthenticated connections.

**CVSS:** 9.1 (CRITICAL) — Unauthenticated real-time event stream; entire platform activity exposed

---

### 12.7 Event Ordering Guarantee

**Test:** Rapidly create 10 capsules in sequence. Observe event delivery order in webhook.

**Expected:** Events delivered in order (capsule_1 → capsule_10).

**Observed:** ⚠️ **PARTIAL** — Events delivered in order within single partition. However, concurrent writes across 3 worker processes cause occasional out-of-order delivery (capsule_7 before capsule_6 in 2/10 runs). No sequence numbers on events.

---

### 12.8 Webhook Payload Size Limit

**Test:** Create a capsule with 500KB content. Observe webhook payload size.

**Expected:** Payload truncated or contains reference URL (not full content).

**Observed:** ✅ **PASS** — Webhook includes only metadata + `data_url` reference. Full content not included in payload. Size capped at ~4KB per webhook delivery.

---

### 12.9 Event Replay & At-Least-Once Delivery

**Test:** Simulate webhook endpoint downtime for 2 hours. Verify events are replayed when endpoint recovers.

**Expected:** Events queued and replayed within retention window (24h).

**Observed:** ✅ **PASS** — Events stored in Redis Streams. Recovery detected via successful 200 response. Queued events replayed in order. 24-hour retention window confirmed.

---

### 12.10 Event Bus Dead Letter Queue

**Test:** Register webhook that always returns 500. Trigger 10 events. Check dead letter queue state.

**Expected:** Events land in DLQ after max retries; admin notification sent.

**Observed:** ⚠️ **PARTIAL** — Events correctly land in DLQ after 5 retries. However, no admin notification is sent (notification hook not implemented). DLQ entries expire after 72 hours without cleanup — can grow unbounded.

---

## Summary

| Test | Status | Severity | CVSS |
|------|--------|----------|------|
| 12.1 Webhook signing | ✅ PASS | — | — |
| 12.2 Retry/backoff | ✅ PASS | — | — |
| 12.3 Cross-tenant leakage | ❌ FAIL | CRITICAL | 8.5 |
| 12.4 SSRF via webhook URL | ❌ FAIL | CRITICAL | 9.0 |
| 12.5 Schema inconsistency | ⚠️ PARTIAL | LOW | 3.0 |
| 12.6 SSE unauthenticated | ❌ FAIL | CRITICAL | 9.1 |
| 12.7 Event ordering | ⚠️ PARTIAL | LOW | 2.0 |
| 12.8 Payload size limit | ✅ PASS | — | — |
| 12.9 Replay/at-least-once | ✅ PASS | — | — |
| 12.10 DLQ notification | ⚠️ PARTIAL | LOW | — |

**Score: 5/10** 🟡 — 3 CRITICAL, 0 HIGH, 0 MEDIUM (but 3 criticals are severe)

---

## Recommended Remediations

### P0 — Emergency (Before Any Event Consumers)
1. **SSE authentication** — Require `Authorization: Bearer` or `?token=` param on `/events/stream` (1h)
2. **Webhook URL allowlist / SSRF block** — Validate webhook URLs against RFC 1918, link-local, metadata CIDRs at registration (2h)
3. **Cross-tenant event routing** — Add `tenant_id` field to event routing key; fan-out only to matching tenant (3h)

### P1 — Sprint 1 (May 1)
4. **Event schema normalization** — Always include `actor` (use `system` for robot-triggered events); standardize `timestamp` to ISO8601 (2h)
5. **DLQ admin alerts** — Wire DLQ threshold alerts to Slack/PagerDuty (1h)

### P2 — Sprint 2
6. **Event sequence numbers** — Add monotonic sequence counter per tenant partition (4h)
7. **DLQ TTL cleanup** — Implement 72h expiry with background job (2h)

---

**T12 Webhooks & Event Streaming — COMPLETE**
**Fig | April 18, 2026 | 2:55 PM PT**
