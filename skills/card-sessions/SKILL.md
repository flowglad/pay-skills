---
name: flowglad-pay-card-sessions
description: Create and redeem Flowglad Pay card sessions to securely access virtual card details (PAN, CVV, expiry). Use this skill when an agent or application needs to retrieve card numbers through the scoped-token security model.
license: MIT
metadata:
  author: flowglad
  version: "1.0.0"
---

# Card Sessions

## Abstract

Card sessions are the secure mechanism for accessing virtual card details (number, CVV, expiry) in Flowglad Pay. A session is created against a payment method, which returns a short-lived scoped JWT token. That token is then used to redeem the session and retrieve card details. This two-step flow ensures card details never leak into MCP tool contexts or long-lived API key scopes.

---

## Table of Contents

1. [Security Model](#1-security-model) — **CRITICAL**
2. [Create a Card Session](#2-create-a-card-session) — **CRITICAL**
3. [Redeem a Card Session](#3-redeem-a-card-session) — **CRITICAL**
4. [Check Session Status](#4-check-session-status) — **MEDIUM**
5. [Audit Redemptions](#5-audit-redemptions) — **MEDIUM**
6. [Session Lifecycle](#6-session-lifecycle) — **LOW**

---

## 1. Security Model

**Impact: CRITICAL — misunderstanding this causes auth failures**

Card details are **never** returned by:
- Standard API key authenticated endpoints
- MCP tools (intentionally omitted from AI context windows)

Instead, card details require a **scoped redeem token** — a short-lived HS256 JWT with:
- `scope`: `card-session:redeem:{paymentMethodId}`
- `sub`: user ID
- Expiry matching the session TTL

The token is returned when creating a session and must be passed via the `X-Scoped-Token` header when redeeming.

---

## 2. Create a Card Session

Creates a session and returns a scoped redeem token.

### REST API

```
POST /v1/card-sessions
Header: X-API-Key: <api-key>
Content-Type: application/json
```

**Request body:**
```json
{
  "paymentMethodId": "pm_abc123",
  "ttlSeconds": 300,
  "maxRedeemCount": 1
}
```

| Field | Type | Required | Default | Validation |
|-------|------|----------|---------|------------|
| `paymentMethodId` | string | Yes | — | Must start with `pm_` |
| `ttlSeconds` | integer | No | 300 | 30–3600 |
| `maxRedeemCount` | integer | No | 1 | 1–10 |

**Response (200):**
```json
{
  "session": {
    "id": "cs_xyz789",
    "userId": "usr_abc",
    "paymentMethodId": "pm_abc123",
    "status": "active",
    "maxRedeemCount": 1,
    "redeemCount": 0,
    "expiresAt": "2026-03-11T12:05:00.000Z",
    "createdAt": "2026-03-11T12:00:00.000Z",
    "updatedAt": "2026-03-11T12:00:00.000Z"
  },
  "redeemToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

### CLI

```bash
fgp card-sessions create <payment-method-id> [--ttl 300] [--max-redemptions 1] [--json]
```

### MCP Tool

```
create_card_session(paymentMethodId, ttlSeconds?, maxRedeemCount?)
```

**Important:** The MCP tool returns session metadata and a hint, but does NOT return card details. You must use the REST API redeem endpoint with the scoped token.

---

## 3. Redeem a Card Session

Exchanges a scoped token for card details. This is the **only** way to get card PAN/CVV/expiry.

### REST API

```
POST /v1/card-sessions/{id}/redeem
Header: X-Scoped-Token: <redeem-token>
Content-Type: application/json
```

**No request body required.** Authentication is via the scoped token only (not the API key).

**Response (200):**
```json
{
  "number": "4242424242424242",
  "expMonth": 12,
  "expYear": 2027,
  "cvc": "123"
}
```

### CLI

```bash
fgp card-sessions redeem <session-id> --token <redeem-token> [--json]
```

### Shortcut: Create + Redeem in One Step

The `fgp card` command creates a session and immediately redeems it:

```bash
fgp card [--payment-method-id pm_abc123] [--json]
```

If using an agent-scoped API key, this automatically resolves the agent's payment method ID.

### Error Responses

| Status | Code | Meaning |
|--------|------|---------|
| 401 | `UNAUTHORIZED` | Missing or invalid scoped token |
| 403 | `FORBIDDEN` | Token scope doesn't match session's payment method |
| 404 | `NOT_FOUND` | Session not found |
| 409 | `CONFLICT` | Session expired, exhausted, or invalid status |

---

## 4. Check Session Status

### REST API

```
GET /v1/card-sessions/{id}
Header: X-API-Key: <api-key>
```

**Response:** Same `session` object as create (without `redeemToken`).

### CLI

```bash
fgp card-sessions get <session-id> [--json]
```

### MCP Tool

```
get_card_session(id)
```

---

## 5. Audit Redemptions

List all redemption records for a session, including IP addresses and timestamps.

### REST API

```
GET /v1/card-sessions/{id}/redemptions
Header: X-API-Key: <api-key>
```

**Response:**
```json
{
  "redemptions": [
    {
      "id": "csr_abc",
      "cardSessionId": "cs_xyz789",
      "ipAddress": "203.0.113.42",
      "redeemedAt": "2026-03-11T12:01:30.000Z"
    }
  ]
}
```

### CLI

```bash
fgp card-sessions redemptions <session-id> [--json]
```

### MCP Tool

```
get_card_session_redemptions(id)
```

---

## 6. Session Lifecycle

Sessions progress through these statuses:

```
active → redeemed    (after maxRedeemCount reached)
active → expired     (after TTL elapses)
redeemed → scrubbed  (vault reference destroyed, card details permanently removed)
expired → scrubbed
```

- **active**: Can be redeemed (redeemCount < maxRedeemCount, not past expiresAt)
- **redeemed**: All redemptions used, card details still in vault temporarily
- **expired**: TTL elapsed, no further redemptions possible
- **scrubbed**: Card details permanently purged from vault
