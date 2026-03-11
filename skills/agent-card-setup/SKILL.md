---
name: flowglad-pay-agent-card-setup
description: Set up a Flowglad Pay virtual card for an AI agent, including onboarding preflight checks, spending policies, and card issuance. Use this skill when onboarding an agent with a payment card or diagnosing why an agent can't access card details.
license: MIT
metadata:
  author: flowglad
  version: "1.0.0"
---

# Agent Card Setup

## Abstract

AI agents in Flowglad Pay can be issued their own virtual cards with spending controls derived from policies. This skill covers the full onboarding flow: checking readiness with `whoami`, understanding spending policies, creating agent cards, and using `fgp card` to retrieve card details.

---

## Table of Contents

1. [Onboarding Preflight (whoami)](#1-onboarding-preflight-whoami) — **CRITICAL**
2. [Onboarding Requirements](#2-onboarding-requirements) — **CRITICAL**
3. [Creating an Agent Card](#3-creating-an-agent-card) — **HIGH**
4. [Retrieving Card Details](#4-retrieving-card-details) — **HIGH**
5. [Spending Policy Composition](#5-spending-policy-composition) — **MEDIUM**
6. [Troubleshooting](#6-troubleshooting) — **MEDIUM**

---

## 1. Onboarding Preflight (whoami)

Check whether an agent is fully onboarded and ready to use card features.

### REST API

```
GET /v1/whoami
Header: X-API-Key: <api-key>
```

**Response (200):**
```json
{
  "userId": "usr_abc",
  "agent": {
    "id": "agt_xyz",
    "name": "My Provisioning Agent",
    "hasPolicies": true,
    "hasPaymentMethod": true,
    "paymentMethodId": "pm_abc123"
  }
}
```

If the API key is not agent-scoped, `agent` is `null`.

### Key Fields

| Field | Meaning |
|-------|---------|
| `agent.hasPolicies` | `true` if the agent has at least one spending policy attached |
| `agent.hasPaymentMethod` | `true` if the agent has a virtual card |
| `agent.paymentMethodId` | The card's payment method ID (used for card sessions) |

---

## 2. Onboarding Requirements

An agent needs three things before it can use card features:

### Step 1: Create a spending limit
Go to `https://app.flowgladpay.com/limits/create` and define:
- **Spend limit**: Maximum amount (e.g., $50.00)
- **Duration**: `one_time`, `monthly`, `quarterly`, or `yearly`
- Optional: merchant restrictions, category restrictions, lock-after date

### Step 2: Attach the limit to the agent
On the agent's detail page in the dashboard, attach the spending limit.

### Step 3: Create a card
On the agent's detail page, create a virtual card. This issues a Brex virtual card with spending controls derived from the attached policies.

**The `whoami` endpoint tells you which steps are missing:**

| `hasPolicies` | `hasPaymentMethod` | Status |
|---------------|---------------------|--------|
| `false` | `false` | Needs steps 1, 2, and 3 |
| `true` | `false` | Needs step 3 only |
| `true` | `true` | Fully onboarded |

---

## 3. Creating an Agent Card

### REST API

```
POST /v1/agents/{id}/card
Header: X-API-Key: <api-key>
```

**No request body.** The endpoint reads the agent's attached policies and creates a card with the composed spending controls.

**Preconditions (all checked server-side):**
1. Agent exists and belongs to the authenticated user
2. Agent has no existing active card
3. Agent has at least one spending policy attached

**Response (200):**
```json
{
  "paymentMethod": {
    "id": "pm_new123",
    "type": "brex",
    "last4": "4242",
    "status": "active",
    "createdAt": 1710158400
  }
}
```

**Error Responses:**

| Code | Meaning |
|------|---------|
| `AGENT_NOT_FOUND` | Agent doesn't exist or doesn't belong to user |
| `CARD_EXISTS` | Agent already has an active card |
| `NO_POLICIES` | No spending policies attached to agent |
| `BREX_ERROR` | Card issuer API failure |

### Getting the Agent's Card

```
GET /v1/agents/{id}/card
Header: X-API-Key: <api-key>
```

Returns the agent's payment method, or 404 if no card exists.

---

## 4. Retrieving Card Details

Once onboarded, use `fgp card` to create a card session and immediately redeem it:

```bash
fgp card [--json]
```

With an agent-scoped API key, this:
1. Calls `GET /v1/whoami` to check onboarding status
2. Validates `hasPolicies` and `hasPaymentMethod`
3. Uses the agent's `paymentMethodId` to create a card session
4. Immediately redeems the session with the scoped token
5. Returns card details (number, expMonth, expYear, cvc)

If any preflight check fails, the CLI prints what steps are still needed instead of erroring.

---

## 5. Spending Policy Composition

When an agent has multiple spending policies, they are composed into a single set of controls using "most restrictive wins":

| Attribute | Composition Rule |
|-----------|-----------------|
| Spend limit | Lowest amount |
| Duration | Shortest period (one_time < monthly < quarterly < yearly) |
| Allowed merchants | Intersection (only merchants in ALL policies) |
| Blocked merchants | Union (merge all blocked lists) |
| Allowed categories | Intersection |
| Blocked categories | Union |
| Lock-after date | Earliest date |

These composed controls are applied to the Brex virtual card at creation time and updated whenever policies are modified.

---

## 6. Troubleshooting

### "Agent has no spending policies"
Attach a spending limit to the agent from the dashboard at `https://app.flowgladpay.com` before creating a card.

### "Agent already has a card"
Each agent can have only one active card. Check with `GET /v1/agents/{id}/card`.

### `fgp card` prints onboarding instructions instead of card details
The `whoami` preflight check detected missing prerequisites. Follow the printed steps.

### Card session redeem returns 403
The scoped token's payment method doesn't match the session. This usually means the token was generated for a different payment method.

### Card session redeem returns 409
The session is expired (past TTL) or exhausted (all redemptions used). Create a new session.
