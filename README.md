 # CasaPay Gateway Integration Skill

> **For AI Agents & Developers:** This document is a language-agnostic skill guide for securely integrating CasaPay Gateway into any application. It covers the full API, security best practices, and common pitfalls.

> **Version:** 1.5 | **Last Updated:** 2026-07-17

---

## Table of Contents

1. [Core Concept](#core-concept)
2. [Environments](#environments)
3. [Authentication](#authentication)
4. [Integration Flow](#integration-flow)
5. [API Reference](#api-reference)
6. [Deposit Modes & Agreement Types](#deposit-modes--agreement-types)
7. [Webhook System](#webhook-system)
8. [Security Requirements (CRITICAL)](#security-requirements-critical)
9. [Error Handling](#error-handling)
10. [Implementation Checklist](#implementation-checklist)
11. [Code Examples (Multi-Language)](#code-examples-multi-language)
12. [Common Pitfalls](#common-pitfalls)
13. [Testing Guide](#testing-guide)

---

## Setup for AI Code Editors

Add this skill to your AI assistant so it can help you integrate CasaPay Gateway correctly.

### Claude Code

```bash
# Add as a skill file
claude skill add https://raw.githubusercontent.com/CasaPay/casapay-gateway-skill/main/README.md
```

Or manually: copy the contents of this file into your project's `CLAUDE.md` or `.claude/skills/` directory.

### Cursor

Add to your project's `.cursor/rules` file:

```
@file https://raw.githubusercontent.com/CasaPay/casapay-gateway-skill/main/README.md
```

Or place this file in your project root as `.cursor/rules/casapay-gateway.md`.

### GitHub Copilot

Add to your `.github/copilot-instructions.md`:

```markdown
## CasaPay Gateway Integration
When implementing CasaPay Gateway integration, follow the guide at:
https://github.com/CasaPay/casapay-gateway-skill
```

### Windsurf / Codeium

Place this file in your project as `.windsurf/rules/casapay-gateway.md` or reference it in your workspace context.

### Any AI Editor (Generic)

1. Download: `curl -O https://raw.githubusercontent.com/CasaPay/casapay-gateway-skill/main/README.md`
2. Place in your project's docs or AI context directory
3. Reference it when asking your AI to implement CasaPay Gateway

---

## Core Concept

CasaPay Gateway is a **Stripe-style hosted checkout** for property/rental payments. It handles:

- **Tenant identity verification** (KYC)
- **Payment collection** (card, bank transfer, open banking)
- **Deposit guarantees** (CasaPay covers deposit, tenant pays monthly subscription)
- **Webhooks** for real-time status updates

### How It Works

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Your Backend    │────▶│ CasaPay API  │────▶│ Gateway Checkout │
│  (create session)│     │              │     │ (hosted page)    │
└─────────────────┘     └──────┬───────┘     └────────┬────────┘
                               │                      │
                               │   Webhook            │  Customer completes
                               │   Confirmation       │  payment/verification
                               ▼                      │
                        ┌──────────────┐              │
                        │  Your Backend │◀─────────────┘
                        │  (webhook)    │    (redirect to success_url)
                        └──────────────┘
```

**Key principle:** The redirect to `success_url` is for the customer's browser. The **webhook** is the authoritative signal that payment/guarantee is complete.

---

## Environments

| Environment | API Base URL | Key Prefix | Purpose |
|-------------|-------------|------------|---------|
| **Production** | Provided during integration | `sk_live_` | Real payments |
| **Sandbox** | Provided during integration | `sk_test_` | Testing |

> **Note:** The exact API base URLs will be provided to you during the integration onboarding process. Store them in environment variables (e.g., `CASAPAY_API_BASE_URL`).

⚠️ **Important:** Sandbox and production use different API keys and URLs. Never use production keys in test environments.

---

## Authentication

All API requests use **Bearer token** authentication:

```
Authorization: Bearer sk_live_YOUR_API_KEY_HERE
```

### Key Management Rules

- Store API keys in **environment variables only** — never in source code
- API keys are **secret** — never expose them in frontend/client-side code
- Each key is tied to a specific **entity** (operator)
- Keys can be **revoked** — implement key rotation support
- `last_used_at` is tracked — monitor for unauthorized use

⚠️ **CRITICAL:** API keys and webhook secrets will be provided during the integration onboarding process.

---

## Integration Flow

### Step-by-Step

```
1. BACKEND:  Create a checkout session via POST /sessions
2. FRONTEND: Redirect customer to the returned `gateway_url`
3. CUSTOMER: Completes checkout (verification + payment) on CasaPay hosted page
4. CASAPAY:  Redirects customer to your `success_url`
5. CASAPAY:  Sends webhook to your `webhook_url` (server-to-server)
6. BACKEND:  Verify webhook signature → confirm payment → fulfill order
7. BACKEND:  (Optional) Poll GET /sessions/{id} as backup confirmation
```

### Option A: Webhook-Based (Recommended)

Use **both** redirect handling AND webhook processing:

```
┌─ Customer Browser ─────────────────────────────────────────┐
│  success_url hit → Show "Processing..." or "Thank you"     │
│  → Poll your own backend for fulfillment status             │
│  → Once confirmed, show final success state                 │
└─────────────────────────────────────────────────────────────┘

┌─ Your Server (Webhook) ───────────────────────────────────┐
│  POST /your-webhook-endpoint                               │
│  → Verify signature                                        │
│  → Verify session status via GET /sessions/{id}            │
│  → Mark order as fulfilled in YOUR database                │
│  → Return 200 OK                                           │
└─────────────────────────────────────────────────────────────┘
```

### Option B: Polling-Based (Simpler, No Webhook Needed)

If you don't want to set up a webhook endpoint, you can poll the session status instead:

```
1. BACKEND:  Create session → get session_id
2. FRONTEND: Redirect customer to gateway_url
3. CUSTOMER: Completes checkout → redirected to success_url
4. BACKEND:  On success_url hit, start polling:
             GET /api/v1/gateway/sessions/{session_id}
             → Repeat every 3-5 seconds until status is terminal
5. BACKEND:  When status == "completed" → fulfill order
```

**Polling pseudocode:**
```
function wait_for_session_completion(session_id, max_attempts=60):
    for attempt in range(max_attempts):
        session = GET /api/v1/gateway/sessions/{session_id}
        
        if session.status == "completed":
            fulfill_order(session)
            return "success"
        
        if session.status in ["expired", "cancelled", "failed"]:
            return session.status  // Terminal — stop polling
        
        sleep(5)  // Wait 5 seconds before next poll
    
    return "timeout"  // Gave up after max attempts
```

**⚠️ Important:** Even with polling, NEVER fulfill an order based solely on the `success_url` redirect. Always confirm via the API first. The redirect can be faked — the API response cannot.

**When to use polling vs webhooks:**

| | Polling | Webhooks |
|---|---|---|
| **Complexity** | Simpler — no endpoint needed | More setup — needs public endpoint |
| **Latency** | 3-5s delay (poll interval) | Near-instant |
| **Reliability** | You control retries | CasaPay handles retries |
| **Best for** | Simple integrations, MVPs | Production, high-volume |

### Option C: Agreement-First (No Immediate Payment)

Use this when you want to **register a tenant relationship first** and send invoices later. No payment session is created upfront — the tenant gets an email alias where invoices can be forwarded.

```
1. BACKEND:  Create agreement via POST /agreements
             → Returns: agreement_id, email_alias (e.g. john.doe+abc@casapay.me)
2. OPERATOR: Forwards rent invoices to the email alias (or uses POST /agreements/{id}/invoice)
3. CASAPAY:  Processes invoice PDF via AI, creates gateway checkout session
4. TENANT:   Receives email with gateway checkout URL → pays via hosted checkout
5. CASAPAY:  Sends webhook to your webhook_url (same as session-first flow)
```

**When to use agreement-first:**

| | Session-First | Agreement-First |
|---|---|---|
| **Use case** | New tenant move-in with immediate payment | Register tenant, send invoices later |
| **Payment timing** | Immediate (at checkout) | When operator sends invoice |
| **Deposit/cover** | Set at session creation | Set at agreement creation |
| **Email alias** | Generated after first payment | Generated immediately |
| **Best for** | Onboarding + payment in one step | Ongoing rent collection via email |

---

## API Reference

### 0. Pricing Preview

```
POST /api/v1/gateway/pricing-preview
Authorization: Bearer sk_live_xxx
Content-Type: application/json
```

Preview the fee breakdown before creating a session. Useful for showing pricing in your UI.

#### Request Body

```json
{
  "cover_amount": 2000.00,
  "first_payment_amount": 900.00,
  "currency": "EUR"
}
```

#### Response `200 OK`

```json
{
  "cover_amount": 2000.00,
  "currency": "EUR",
  "deposit_options": [
    {
      "mode": "deposit_upfront",
      "deposit_amount": 2000.00,
      "first_payment_amount": 900.00,
      "due_today": 2900.00,
      "subscription": null
    },
    {
      "mode": "deposit_guaranteed",
      "deposit_amount": 0.00,
      "first_payment_amount": 900.00,
      "due_today": 912.00,
      "subscription": {
        "monthly_amount": 12.00,
        "annual_amount": 120.00,
        "coverage_amount": 2000.00
      }
    }
  ]
}
```

---

### 1. Create Checkout Session

```
POST /api/v1/gateway/sessions
Authorization: Bearer sk_live_xxx
Content-Type: application/json
```

#### Request Body

```json
{
  "tenant": {
    "email": "john@example.com",
    "first_name": "John",
    "last_name": "Doe",
    "phone": "+3725551234",
    "personal_code": "39001011234"
  },
  "cover_amount": 2000.00,
  "deposit_mode": "choice",
  "first_payment_amount": 900.00,
  "first_payment_description": "First month's rent - April 2026",
  "currency": "EUR",
  "description": "Move-in payment for Apartment 4B",
  "document_url": "https://example.com/contract.pdf",
  "verification_required": null,
  "allowed_payment_methods": ["card", "bank_transfer"],
  "success_url": "https://yourapp.com/payment/success?session={session_id}",
  "cancel_url": "https://yourapp.com/payment/cancel",
  "metadata": {
    "order_id": "ORD-12345",
    "property_id": "PROP-789"
  }
}
```

#### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tenant.email` | string | ✅ | Customer email |
| `tenant.first_name` | string | ✅ | Customer first name |
| `tenant.last_name` | string | ✅ | Customer last name |
| `tenant.phone` | string | ❌ | Customer phone (E.164 format) |
| `tenant.personal_code` | string | ❌ | Customer personal/ID code |
| `cover_amount` | number | ❌* | Deposit/guarantee amount (max: €15,000) |
| `deposit_mode` | string | Required with `cover_amount` | `deposit_upfront`, `deposit_guaranteed`, or `choice` |
| `agreement_type` | string | ❌ | `ontime` (default), `cover`, or `payment_link`. Controls what PaymentAgreement type is created. `ontime` = immediate payout on due date. `cover` = payout delayed 30 days |
| `first_payment_amount` | number | ❌* | First rent/payment amount |
| `first_payment_description` | string | ❌ | Description for first payment line |
| `amount` | number | ❌* | Legacy: single payment amount (no deposit split) |
| `currency` | string | ❌ | `EUR` or `GBP` (default: `EUR`) |
| `description` | string | ❌ | Description shown on checkout page |
| `document_url` | string | ❌ | URL to a document (contract, etc.) |
| `verification_required` | bool/null | ❌ | `true`=force, `false`=skip, `null`=entity default |
| `allowed_payment_methods` | array | ❌ | `["card", "bank_transfer", "open_banking"]` |
| `success_url` | string | ✅ | Redirect URL after success. `{session_id}` placeholder supported |
| `cancel_url` | string | ✅ | Redirect URL if customer cancels |
| `metadata` | object | ❌ | Arbitrary key-value pairs (returned in webhooks, max 20 keys) |

*At least one of `cover_amount`, `first_payment_amount`, or `amount` is required.

#### Response `201 Created`

```json
{
  "session_id": "gwy_abc123def456ghi789jkl012",
  "gateway_url": "https://gateway.casapay.com/gwy_abc123def456ghi789jkl012",
  "status": "pending",
  "expires_at": "2026-03-12T12:00:00+00:00",
  "cover_amount": 2000.00,
  "deposit_mode": "choice",
  "first_payment_amount": 900.00,
  "first_payment_description": "First month's rent - April 2026",
  "total_amount": 2900.00
}
```

**After receiving this response, redirect the customer's browser to `gateway_url`.**

---

### 1b. Create Agreement (Agreement-First, No Immediate Payment)

```
POST /api/v1/gateway/agreements
Authorization: Bearer sk_live_xxx
Content-Type: application/json
```

Creates a PaymentAgreement with an email alias **without requiring a payment session**. The operator registers a tenant relationship upfront, then sends invoices later via the email alias or the invoice endpoint.

#### Request Body

```json
{
  "tenant": {
    "email": "john@example.com",
    "first_name": "John",
    "last_name": "Doe",
    "phone": "+3725551234",
    "personal_code": "39001011234"
  },
  "agreement_type": "payment_link",
  "cover_amount": 5000.00,
  "currency": "EUR",
  "description": "Apartment 5B, Tallinn"
}
```

#### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tenant.email` | string | ✅ | Tenant email (find-or-create) |
| `tenant.first_name` | string | ✅ | Tenant first name |
| `tenant.last_name` | string | ✅ | Tenant last name |
| `tenant.phone` | string | ❌ | Phone (E.164 format) |
| `tenant.personal_code` | string | ❌ | Personal/ID code |
| `agreement_type` | string | ✅ | `payment_link`, `ontime`, or `cover` |
| `cover_amount` | number | ❌ | Guarantee/cover amount. Only for `ontime`/`cover`. Validated against entity `max_cover_amount`. |
| `currency` | string | ❌ | `EUR` or `GBP` (default: entity currency) |
| `description` | string | ❌ | Agreement title / description |

#### Agreement Type Behavior

| Type | Cover Amount | Email Alias Invoices | Payout |
|------|-------------|---------------------|--------|
| `payment_link` | ❌ Not allowed | Creates payment link | Immediate |
| `ontime` | ✅ Optional | Creates gateway checkout session | Immediate (due date) |
| `cover` | ✅ Optional | Creates gateway checkout session | Due date + 30 days |

#### Response `201 Created`

```json
{
  "id": 123,
  "agreement_id": "PA-20260417-ABC123",
  "agreement_type": "payment_link",
  "status": "active",
  "email_alias": "johndoe-abc123@casapay.me",
  "currency": "EUR",
  "cover_amount": null,
  "description": "Apartment 5B, Tallinn",
  "tenant": {
    "id": 456,
    "email": "john@example.com",
    "first_name": "John",
    "last_name": "Doe"
  },
  "created_at": "2026-04-17T00:00:00+00:00"
}
```

**After creating the agreement, the operator can:**
1. Forward invoice PDFs to the `email_alias` — CasaPay AI extracts the amount and sends the tenant a payment link (for `payment_link`) or gateway checkout URL (for `ontime`/`cover`)
2. Use `POST /agreements/{id}/invoice` to create payment sessions programmatically

---

### 2. Create Follow-Up Invoice Session

```
POST /api/v1/gateway/agreements/{paymentAgreementId}/invoice
Authorization: Bearer sk_live_xxx
Content-Type: application/json
```

Creates a payment-only session on an existing active PaymentAgreement. No tenant data, verification, or deposit needed — the tenant just pays.

#### Request Body

```json
{
  "amount": 900.00,
  "description": "Monthly rent - May 2026",
  "document_url": "https://example.com/invoice-may.pdf",
  "success_url": "https://yourapp.com/payment/success",
  "cancel_url": "https://yourapp.com/payment/cancel",
  "metadata": { "invoice_number": "INV-2026-05" }
}
```

#### Response `201 Created`

```json
{
  "session_id": "gwy_xyz789",
  "gateway_url": "https://gateway.casapay.com/gwy_xyz789",
  "status": "pending",
  "amount": 900.00,
  "currency": "EUR",
  "description": "Monthly rent - May 2026",
  "payment_agreement_id": 123,
  "tenant": {
    "email": "john@example.com",
    "first_name": "John",
    "last_name": "Doe"
  },
  "expires_at": "2026-04-12T12:00:00+00:00"
}
```

---

### 3. Get Agreement Details

```
GET /api/v1/gateway/agreements/{paymentAgreementId}
Authorization: Bearer sk_live_xxx
```

Retrieve details of a payment agreement created via the gateway. Only returns agreements belonging to your entity.

#### Response `200 OK`

```json
{
  "id": 123,
  "agreement_id": "PA-20260311-ABC123",
  "agreement_title": "Apartment 4B - John Doe",
  "agreement_type": "ontime",
  "status": "active",
  "email_alias": "apt4b@casapay.me",
  "currency": "EUR",
  "coverage_amount": 2000.00,
  "agreement_start": "2026-03-11",
  "agreement_end": "2027-03-11",
  "tenant": {
    "id": 42,
    "email": "john@example.com",
    "first_name": "John",
    "last_name": "Doe"
  },
  "created_at": "2026-03-11T12:00:00+00:00",
  "updated_at": "2026-03-11T14:30:00+00:00"
}
```

| Field | Description |
|-------|-------------|
| `email_alias` | CasaPay email forwarding address (e.g. `apt4b@casapay.me`) — tenants send invoices here |
| `coverage_amount` | The deposit/guarantee amount CasaPay covers (`null` for payment_link agreements) |
| `agreement_type` | `ontime`, `cover`, or `payment_link` |
| `status` | `draft`, `active`, `terminated`, etc. |

---

### 4. Get Session Status

```
GET /api/v1/gateway/sessions/{session_id}
Authorization: Bearer sk_live_xxx
```

#### Response `200 OK`

```json
{
  "session_id": "gwy_abc123def456ghi789jkl012",
  "status": "completed",
  "current_step": null,
  "amount": 2900.00,
  "currency": "EUR",
  "description": "Move-in payment for Apartment 4B",
  "payment_agreement_id": 123,
  "cover": {
    "amount": 2000.00,
    "mode": "choice",
    "resolved_mode": "deposit_guaranteed"
  },
  "first_payment": {
    "amount": 900.00,
    "description": "First month's rent - April 2026"
  },
  "total_cash_amount": 900.00,
  "total_guaranteed_amount": 2000.00,
  "tenant": {
    "id": 42,
    "email": "john@example.com",
    "first_name": "John",
    "last_name": "Doe",
    "verification_status": "verified"
  },
  "payment": {
    "status": "paid",
    "paid_at": "2026-03-11T14:30:00+00:00",
    "payment_method": "card",
    "transaction_id": "txn_abc123"
  },
  "guarantee": {
    "status": "active",
    "guarantee_id": "GRN-20260311-ABC123",
    "covered_amount": 2000.00,
    "activated_at": "2026-03-11T14:28:00+00:00"
  },
  "metadata": { "order_id": "ORD-12345" },
  "created_at": "2026-03-11T12:00:00+00:00",
  "completed_at": "2026-03-11T14:30:00+00:00"
}
```

#### Session Statuses

| Status | Description | Is Terminal? |
|--------|-------------|-------------|
| `pending` | Created, waiting for customer | No |
| `processing` | Customer actively completing steps | No |
| `completed` | All steps done, payment successful | ✅ Yes |
| `expired` | Timed out (default: 24h) | ✅ Yes |
| `cancelled` | Cancelled by customer or operator | ✅ Yes |
| `failed` | Payment or verification failed | ✅ Yes |

---

### 4. Cancel Session

```
POST /api/v1/gateway/sessions/{session_id}/cancel
Authorization: Bearer sk_live_xxx
```

Response: `200 OK` with `{"session_id": "...", "status": "cancelled"}`

---

## Deposit Modes & Agreement Types

### Deposit Modes

The `deposit_mode` field controls how the security deposit is handled:

| Mode | What Customer Sees | What Happens |
|------|-------------------|--------------|
| `deposit_upfront` | Pays full deposit amount upfront | Traditional deposit payment |
| `deposit_guaranteed` | KYC verification → CasaPay Guarantee activated | No deposit paid; CasaPay guarantees it |
| `choice` | **Two options:** pay upfront OR use CasaPay Guarantee | Customer decides on checkout page |

### How `choice` Mode Works

When `deposit_mode` is `choice`, the checkout page shows two options:

- **Option A (Upfront):** Pay `cover_amount + first_payment_amount` = total
- **Option B (Guaranteed):** Pay only `first_payment_amount` + monthly guarantee fee (~0.6%/month of cover)

The customer's choice is recorded as `resolved_mode` in the session (`deposit_upfront` or `deposit_guaranteed`).

### Use the Pricing Preview First

Before creating a session, call `POST /pricing-preview` with `cover_amount` and `first_payment_amount` to get the exact fee breakdown for both options. Display this in your UI so the tenant knows what to expect.

### Agreement Types (Internal)

Behind the scenes, CasaPay creates different agreement types:

| Type | Description | Created When | Payout Timing |
|------|-------------|-------------|---------------|
| `ontime` | Proactive guarantee — CasaPay pays operator on invoice due date | `agreement_type: "ontime"` (default) | Immediate (due date) |
| `cover` | Reactive guarantee — CasaPay pays after 30-day grace period | `agreement_type: "cover"` | Due date + 30 days |
| `payment_link` | No guarantee — just payment forwarding to operator | `agreement_type: "payment_link"` | Immediate (due date) |

> **Note:** `guarantor` is a legacy type still present in the system for existing agreements but is **not accepted** by the Gateway API. Use `ontime` instead.

---

## Webhook System

### Event Types

| Event | Fired When |
|-------|-----------|
| `gateway.session.completed` | All steps done (paid + optionally guaranteed) |
| `gateway.session.expired` | Session timed out |
| `gateway.session.cancelled` | Session cancelled |
| `gateway.verification.failed` | KYC verification failed |
| `gateway.payment.failed` | Payment attempt failed |
| `gateway.guarantee.activated` | CasaPay Guarantee activated for deposit |

### Webhook Payload Example (`gateway.session.completed`)

```json
{
  "event": "gateway.session.completed",
  "session_id": "gwy_abc123def456ghi789jkl012",
  "timestamp": "2026-03-11T14:30:00+00:00",
  "tenant": {
    "id": 42,
    "email": "john@example.com",
    "first_name": "John",
    "last_name": "Doe",
    "verification_status": "verified"
  },
  "payment": {
    "cash_amount": 900.00,
    "guaranteed_amount": 2000.00,
    "total_amount": 2900.00,
    "currency": "EUR",
    "payment_method": "card",
    "paid_at": "2026-03-11T14:30:00+00:00",
    "transaction_id": "txn_abc123"
  },
  "deposit": {
    "amount": 2000.00,
    "mode": "choice",
    "resolved_mode": "guarantee"
  },
  "guarantee": {
    "guarantee_id": "GRN-20260311-ABC123",
    "covered_amount": 2000.00,
    "status": "active",
    "activated_at": "2026-03-11T14:28:00+00:00"
  },
  "metadata": { "order_id": "ORD-12345" }
}
```

### Webhook Headers

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `User-Agent` | `CasaPay-Gateway/1.0` |
| `X-CasaPay-Timestamp` | Unix timestamp (seconds) |
| `X-CasaPay-Signature` | `sha256=<HMAC-SHA256 hex digest>` |

### Signature Verification (CRITICAL)

The signature is computed as:

```
HMAC-SHA256(webhook_secret, timestamp + "." + raw_json_body)
```

**Pseudocode:**

```
function verify_webhook(raw_body, timestamp, signature_header, webhook_secret):
    // 1. Extract the hex digest from the header
    received_sig = signature_header.replace("sha256=", "")
    
    // 2. Compute the expected signature
    message = timestamp + "." + raw_body
    expected_sig = hmac_sha256(webhook_secret, message).hex()
    
    // 3. Use constant-time comparison to prevent timing attacks
    return constant_time_equal(received_sig, expected_sig)
```

### Retry Policy

CasaPay retries failed webhooks (non-2xx responses or timeouts) automatically:

| Attempt | Delay After Failure |
|---------|-------------------|
| 1 | Immediate |
| 2 | +5 minutes |
| 3 | +30 minutes |
| 4 | +2 hours |
| 5 | +6 hours |
| 6 | +24 hours |
| 7+ | Gives up (max 6 attempts) |

**Your webhook endpoint MUST:**
- Return `200 OK` quickly (within 30 seconds)
- Be idempotent (same webhook delivered multiple times = same result)
- Process asynchronously if heavy work is needed

---

## Security Requirements (CRITICAL)

### 🔴 Rule #1: NEVER Trust Return URLs Alone

When a customer completes checkout, CasaPay redirects them to your `success_url`. This is a **browser redirect** and can be trivially faked by an attacker visiting the URL directly.

```
❌ INSECURE (NEVER DO THIS):
  GET /success?session=gwy_abc123
    → Mark tenant as verified
    → Grant access
    → Enable services

✅ SECURE (ALWAYS DO THIS):
  GET /success?session=gwy_abc123
    → Show "Processing your payment..."
    → Frontend polls YOUR backend for status
    → Backend only updates after webhook verification
```

### 🔴 Rule #2: ALWAYS Verify Webhook Signatures

Every webhook includes an HMAC-SHA256 signature. You MUST verify it before processing:

```
function handle_webhook(request):
    raw_body = request.raw_body()  // NOT parsed JSON — the raw string
    timestamp = request.header("X-CasaPay-Timestamp")
    signature = request.header("X-CasaPay-Signature")
    
    // Reject if timestamp is too old (prevent replay attacks)
    if abs(current_time() - int(timestamp)) > 300:  // 5 minute tolerance
        return 401, "Timestamp too old"
    
    // Verify HMAC signature
    if not verify_webhook(raw_body, timestamp, signature, WEBHOOK_SECRET):
        return 401, "Invalid signature"
    
    // NOW safe to process
    event = parse_json(raw_body)
    process_event(event)
    return 200, "OK"
```

### 🔴 Rule #3: Double-Check with API Call

After receiving a webhook, make a **server-side API call** to confirm the session status:

```
function process_completed_webhook(event):
    session_id = event["session_id"]
    
    // Double-check: fetch session from API
    session = GET /api/v1/gateway/sessions/{session_id}
    
    if session.status != "completed":
        log_warning("Webhook says completed but API says: " + session.status)
        return  // Don't fulfill
    
    // NOW it's safe to fulfill
    fulfill_order(session)
```

### 🔴 Rule #4: Idempotent Webhook Processing

CasaPay may send the same webhook multiple times (retries). Your handler MUST be idempotent:

```
function handle_completed_event(event):
    session_id = event["session_id"]
    
    // Check if already processed
    if database.exists("processed_webhooks", session_id):
        return 200  // Already handled, return success
    
    // Process the event
    fulfill_order(event)
    
    // Mark as processed
    database.insert("processed_webhooks", {
        session_id: session_id,
        processed_at: now(),
        event_type: event["event"]
    })
    
    return 200
```

### 🔴 Rule #5: Protect API Keys

| Do | Don't |
|----|-------|
| Store in environment variables | Hardcode in source code |
| Use server-side only | Expose in frontend/client code |
| Use separate keys per environment | Use production keys in development |
| Implement key rotation | Use one key forever |
| Log usage anomalies | Ignore `last_used_at` |

### 🔴 Rule #6: HTTPS Only

All communication with CasaPay API MUST be over HTTPS. Reject any non-HTTPS webhook URLs.

### 🔴 Rule #7: Key Provisioning

API keys and webhook secrets are:
- Provisioned by CasaPay during onboarding
- Rotatable via secure request to CasaPay support

---

## Error Handling

### HTTP Error Codes

| Code | Error | Description | Action |
|------|-------|-------------|--------|
| `401` | `MISSING_API_KEY` | No Bearer token in request | Add `Authorization` header |
| `401` | `INVALID_API_KEY` | Key not found or revoked | Check key, request new one |
| `403` | `GATEWAY_NOT_ENABLED` | Gateway not enabled for this entity | Contact CasaPay |
| `404` | `SESSION_NOT_FOUND` | Session ID doesn't exist | Verify session_id |
| `400` | `SESSION_NOT_ACTIVE` | Session in terminal state | Create new session |
| `400` | `GUARANTEE_LIMIT_EXCEEDED` | Deposit > €15,000 | Reduce deposit amount |
| `422` | `VALIDATION_ERROR` | Request body validation failed | Check error details |
| `429` | (Rate limited) | Too many requests | Implement backoff |

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The given data was invalid.",
    "details": {
      "tenant.email": ["The tenant.email field is required."],
      "success_url": ["The success url field is required."]
    }
  }
}
```

### Recommended Error Handling Pattern

```
function create_session(session_data):
    try:
        response = POST /api/v1/gateway/sessions, body=session_data
        
        if response.status == 201:
            return response.json()  // Success
        
        if response.status == 422:
            // Validation error — fix the request
            log_error("Validation failed", response.json().error.details)
            raise ValidationError(response.json())
        
        if response.status == 401:
            // Auth error — check API key
            log_critical("API key invalid or missing")
            raise AuthError()
        
        if response.status == 429:
            // Rate limited — wait and retry
            retry_after = response.header("Retry-After") or 60
            sleep(retry_after)
            return create_session(session_data)  // Retry
        
        // Unexpected error
        log_error("Unexpected API error", response.status, response.body)
        raise ApiError(response.status)
        
    catch NetworkError:
        // Network issue — retry with backoff
        log_warning("Network error, retrying...")
        sleep(exponential_backoff())
        return create_session(session_data)
```

---

## Implementation Checklist

### Backend Setup
- [ ] Store `CASAPAY_API_KEY` in environment variables
- [ ] Store `CASAPAY_WEBHOOK_SECRET` in environment variables
- [ ] Store `CASAPAY_API_BASE_URL` in environment variables
- [ ] Create database table/column for `gateway_session_id` on your order/lease model
- [ ] Create database table for `processed_webhooks` (idempotency tracking)

### Session Creation
- [ ] Implement `POST /sessions` call from your backend
- [ ] Store returned `session_id` in your database linked to the order
- [ ] Redirect customer to `gateway_url`
- [ ] Handle all error responses (401, 403, 422, etc.)

### Success Page
- [ ] Create success page at your `success_url`
- [ ] Show "Processing..." state (NOT "Payment confirmed!")
- [ ] Frontend polls YOUR backend for fulfillment status
- [ ] Only show confirmed state after YOUR backend confirms

### Webhook Handler
- [ ] Create webhook endpoint (e.g., `POST /webhooks/casapay`)
- [ ] Parse raw body (before JSON parsing) for signature verification
- [ ] Verify `X-CasaPay-Signature` using HMAC-SHA256
- [ ] Verify `X-CasaPay-Timestamp` is within 5 minutes
- [ ] Check idempotency (skip if already processed)
- [ ] Verify session status via `GET /sessions/{id}` API call
- [ ] Fulfill order / activate tenant
- [ ] Return `200 OK` quickly
- [ ] Log all webhook receipts for debugging

### Cancel Page
- [ ] Create cancel page at your `cancel_url`
- [ ] Allow customer to retry (create new session)

### Monitoring
- [ ] Log all API calls and responses
- [ ] Monitor webhook delivery success rate
- [ ] Alert on signature verification failures (potential attack)
- [ ] Alert on unmatched session IDs
- [ ] Monitor for sessions that complete but webhook never arrives (use polling as fallback)

---

## Code Examples (Multi-Language)

### Pseudocode (Reference Implementation)

```
// ===== SESSION CREATION =====

function create_checkout_session(order):
    response = http_post(
        url: ENV["CASAPAY_API_BASE_URL"] + "/sessions",
        headers: {
            "Authorization": "Bearer " + ENV["CASAPAY_API_KEY"],
            "Content-Type": "application/json"
        },
        body: {
            "tenant": {
                "email": order.customer_email,
                "first_name": order.customer_first_name,
                "last_name": order.customer_last_name
            },
            "deposit_amount": order.deposit_amount,
            "deposit_mode": "choice",
            "first_payment_amount": order.first_payment,
            "first_payment_description": order.payment_description,
            "success_url": APP_URL + "/payments/success?session={session_id}",
            "cancel_url": APP_URL + "/payments/cancel",
            "metadata": {
                "order_id": order.id
            }
        }
    )
    
    if response.status != 201:
        raise Error("Failed to create session: " + response.body)
    
    data = response.json()
    
    // Store session reference
    order.gateway_session_id = data["session_id"]
    order.gateway_status = "pending"
    order.save()
    
    return data["gateway_url"]


// ===== WEBHOOK HANDLER =====

function handle_casapay_webhook(request):
    raw_body = request.raw_body()
    timestamp = request.header("X-CasaPay-Timestamp")
    signature = request.header("X-CasaPay-Signature")
    
    // Step 1: Validate timestamp (anti-replay)
    if not timestamp or abs(now_unix() - int(timestamp)) > 300:
        return response(401, "Invalid timestamp")
    
    // Step 2: Verify signature
    expected = "sha256=" + hmac_sha256(ENV["CASAPAY_WEBHOOK_SECRET"], timestamp + "." + raw_body).hex()
    if not constant_time_equal(signature, expected):
        log_security("Invalid webhook signature from " + request.ip)
        return response(401, "Invalid signature")
    
    // Step 3: Parse event
    event = json_parse(raw_body)
    session_id = event["session_id"]
    
    // Step 4: Idempotency check
    if database.exists("processed_webhooks", where: session_id = session_id, event = event["event"]):
        return response(200, "Already processed")
    
    // Step 5: Handle event
    switch event["event"]:
        case "gateway.session.completed":
            handle_session_completed(event)
        case "gateway.session.expired":
            handle_session_expired(event)
        case "gateway.session.cancelled":
            handle_session_cancelled(event)
        case "gateway.payment.failed":
            handle_payment_failed(event)
    
    // Step 6: Record processing
    database.insert("processed_webhooks", {
        session_id: session_id,
        event_type: event["event"],
        processed_at: now()
    })
    
    return response(200, "OK")


function handle_session_completed(event):
    session_id = event["session_id"]
    
    // CRITICAL: Verify with API call
    api_session = http_get(
        url: ENV["CASAPAY_API_BASE_URL"] + "/sessions/" + session_id,
        headers: { "Authorization": "Bearer " + ENV["CASAPAY_API_KEY"] }
    ).json()
    
    if api_session["status"] != "completed":
        log_warning("API status mismatch", session_id, api_session["status"])
        return
    
    // Find the order
    order = database.find("orders", where: gateway_session_id = session_id)
    if not order:
        log_error("No order found for session", session_id)
        return
    
    // Update order
    order.gateway_status = "completed"
    order.paid_at = event["payment"]["paid_at"]
    order.payment_method = event["payment"]["payment_method"]
    order.cash_amount = event["payment"]["cash_amount"]
    order.guaranteed_amount = event["payment"]["guaranteed_amount"]
    
    if event["guarantee"]:
        order.guarantee_id = event["guarantee"]["guarantee_id"]
        order.guarantee_status = event["guarantee"]["status"]
    
    order.save()
    
    // Trigger downstream actions
    send_confirmation_email(order)
    activate_tenant(order)
```

### Node.js (Express)

```javascript
const crypto = require('crypto');
const express = require('express');
const app = express();

const CASAPAY_API_KEY = process.env.CASAPAY_API_KEY;
const CASAPAY_WEBHOOK_SECRET = process.env.CASAPAY_WEBHOOK_SECRET;
const CASAPAY_API_URL = process.env.CASAPAY_API_URL || 'https://api.casapay.com/api/v1/gateway';

// ===== CREATE SESSION =====
app.post('/create-checkout', async (req, res) => {
  try {
    const response = await fetch(`${CASAPAY_API_URL}/sessions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${CASAPAY_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        tenant: {
          email: req.body.email,
          first_name: req.body.firstName,
          last_name: req.body.lastName,
        },
        deposit_amount: req.body.depositAmount,
        deposit_mode: 'choice',
        first_payment_amount: req.body.firstPayment,
        first_payment_description: req.body.paymentDescription,
        success_url: `${process.env.APP_URL}/payment/success?session={session_id}`,
        cancel_url: `${process.env.APP_URL}/payment/cancel`,
        metadata: { order_id: req.body.orderId },
      }),
    });

    if (!response.ok) {
      const error = await response.json();
      return res.status(response.status).json({ error });
    }

    const data = await response.json();

    // Store session_id linked to order
    await db.orders.update(req.body.orderId, {
      gateway_session_id: data.session_id,
      gateway_status: 'pending',
    });

    res.json({ gateway_url: data.gateway_url });
  } catch (err) {
    console.error('CasaPay session creation failed:', err);
    res.status(500).json({ error: 'Payment initialization failed' });
  }
});

// ===== WEBHOOK HANDLER =====
// IMPORTANT: Use raw body parser for webhooks
app.post('/webhooks/casapay', express.raw({ type: 'application/json' }), async (req, res) => {
  const rawBody = req.body.toString('utf8');
  const timestamp = req.headers['x-casapay-timestamp'];
  const signature = req.headers['x-casapay-signature'];

  // 1. Validate timestamp
  if (!timestamp || Math.abs(Date.now() / 1000 - parseInt(timestamp)) > 300) {
    console.warn('Webhook rejected: invalid timestamp');
    return res.status(401).json({ error: 'Invalid timestamp' });
  }

  // 2. Verify signature
  const expectedSig = 'sha256=' + crypto
    .createHmac('sha256', CASAPAY_WEBHOOK_SECRET)
    .update(`${timestamp}.${rawBody}`)
    .digest('hex');

  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expectedSig))) {
    console.error('Webhook rejected: invalid signature from', req.ip);
    return res.status(401).json({ error: 'Invalid signature' });
  }

  // 3. Parse and process
  const event = JSON.parse(rawBody);

  // 4. Idempotency check
  const existing = await db.processedWebhooks.findOne({ 
    session_id: event.session_id, 
    event_type: event.event 
  });
  if (existing) {
    return res.json({ status: 'already_processed' });
  }

  // 5. Handle events
  try {
    switch (event.event) {
      case 'gateway.session.completed':
        await handleSessionCompleted(event);
        break;
      case 'gateway.session.expired':
      case 'gateway.session.cancelled':
        await handleSessionTerminated(event);
        break;
      case 'gateway.payment.failed':
        await handlePaymentFailed(event);
        break;
    }

    // 6. Record
    await db.processedWebhooks.create({
      session_id: event.session_id,
      event_type: event.event,
      processed_at: new Date(),
    });

    res.json({ received: true });
  } catch (err) {
    console.error('Webhook processing error:', err);
    res.status(500).json({ error: 'Processing failed' });
  }
});

async function handleSessionCompleted(event) {
  // CRITICAL: Double-check with API
  const apiResponse = await fetch(
    `${CASAPAY_API_URL}/sessions/${event.session_id}`,
    { headers: { 'Authorization': `Bearer ${CASAPAY_API_KEY}` } }
  );
  const session = await apiResponse.json();

  if (session.status !== 'completed') {
    console.warn(`API status mismatch: ${session.status} for ${event.session_id}`);
    return;
  }

  // Update order
  const order = await db.orders.findOne({ gateway_session_id: event.session_id });
  if (!order) {
    console.error(`No order for session ${event.session_id}`);
    return;
  }

  await db.orders.update(order.id, {
    gateway_status: 'completed',
    paid_at: event.payment.paid_at,
    cash_amount: event.payment.cash_amount,
    guaranteed_amount: event.payment.guaranteed_amount,
    guarantee_id: event.guarantee?.guarantee_id || null,
  });

  // Trigger downstream
  await sendConfirmationEmail(order);
  await activateTenant(order);
}
```

### Python (Flask)

```python
import hmac
import hashlib
import time
import json
import requests
from flask import Flask, request, jsonify
import os

app = Flask(__name__)

CASAPAY_API_KEY = os.environ['CASAPAY_API_KEY']
CASAPAY_WEBHOOK_SECRET = os.environ['CASAPAY_WEBHOOK_SECRET']
CASAPAY_API_URL = os.environ.get('CASAPAY_API_URL', 'https://api.casapay.com/api/v1/gateway')


def verify_webhook_signature(raw_body: bytes, timestamp: str, signature: str) -> bool:
    """Verify CasaPay webhook HMAC-SHA256 signature."""
    message = f"{timestamp}.{raw_body.decode('utf-8')}"
    expected = 'sha256=' + hmac.new(
        CASAPAY_WEBHOOK_SECRET.encode('utf-8'),
        message.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(signature, expected)


@app.route('/create-checkout', methods=['POST'])
def create_checkout():
    data = request.json
    
    response = requests.post(
        f"{CASAPAY_API_URL}/sessions",
        headers={
            'Authorization': f'Bearer {CASAPAY_API_KEY}',
            'Content-Type': 'application/json',
        },
        json={
            'tenant': {
                'email': data['email'],
                'first_name': data['first_name'],
                'last_name': data['last_name'],
            },
            'deposit_amount': data.get('deposit_amount'),
            'deposit_mode': 'choice',
            'first_payment_amount': data.get('first_payment'),
            'success_url': f"{os.environ['APP_URL']}/payment/success?session={{session_id}}",
            'cancel_url': f"{os.environ['APP_URL']}/payment/cancel",
            'metadata': {'order_id': data['order_id']},
        }
    )
    
    if response.status_code != 201:
        return jsonify({'error': response.json()}), response.status_code
    
    session = response.json()
    
    # Store in your database
    db.orders.update(data['order_id'], 
                     gateway_session_id=session['session_id'],
                     gateway_status='pending')
    
    return jsonify({'gateway_url': session['gateway_url']})


@app.route('/webhooks/casapay', methods=['POST'])
def casapay_webhook():
    raw_body = request.get_data()
    timestamp = request.headers.get('X-CasaPay-Timestamp', '')
    signature = request.headers.get('X-CasaPay-Signature', '')
    
    # 1. Validate timestamp
    try:
        if abs(time.time() - int(timestamp)) > 300:
            return jsonify({'error': 'Timestamp too old'}), 401
    except (ValueError, TypeError):
        return jsonify({'error': 'Invalid timestamp'}), 401
    
    # 2. Verify signature
    if not verify_webhook_signature(raw_body, timestamp, signature):
        app.logger.error(f"Invalid webhook signature from {request.remote_addr}")
        return jsonify({'error': 'Invalid signature'}), 401
    
    # 3. Parse event
    event = json.loads(raw_body)
    session_id = event['session_id']
    event_type = event['event']
    
    # 4. Idempotency
    if db.processed_webhooks.exists(session_id=session_id, event_type=event_type):
        return jsonify({'status': 'already_processed'}), 200
    
    # 5. Handle event
    if event_type == 'gateway.session.completed':
        _handle_completed(event)
    elif event_type in ('gateway.session.expired', 'gateway.session.cancelled'):
        _handle_terminated(event)
    
    # 6. Record
    db.processed_webhooks.create(
        session_id=session_id,
        event_type=event_type,
        processed_at=datetime.utcnow()
    )
    
    return jsonify({'received': True}), 200


def _handle_completed(event):
    session_id = event['session_id']
    
    # CRITICAL: Double-check via API
    resp = requests.get(
        f"{CASAPAY_API_URL}/sessions/{session_id}",
        headers={'Authorization': f'Bearer {CASAPAY_API_KEY}'}
    )
    api_session = resp.json()
    
    if api_session.get('status') != 'completed':
        app.logger.warning(f"Status mismatch for {session_id}: {api_session.get('status')}")
        return
    
    order = db.orders.find_one(gateway_session_id=session_id)
    if not order:
        app.logger.error(f"No order for session {session_id}")
        return
    
    order.update(
        gateway_status='completed',
        paid_at=event['payment']['paid_at'],
        cash_amount=event['payment']['cash_amount'],
        guaranteed_amount=event['payment']['guaranteed_amount'],
        guarantee_id=event.get('guarantee', {}).get('guarantee_id'),
    )
    
    send_confirmation_email(order)
    activate_tenant(order)
```

### PHP (Laravel)

```php
<?php
// routes/web.php
Route::post('/webhooks/casapay', [CasaPayWebhookController::class, 'handle'])
    ->withoutMiddleware([\App\Http\Middleware\VerifyCsrfToken::class]);

// app/Http/Controllers/CasaPayWebhookController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class CasaPayWebhookController extends Controller
{
    public function handle(Request $request)
    {
        $rawBody = $request->getContent();
        $timestamp = $request->header('X-CasaPay-Timestamp');
        $signature = $request->header('X-CasaPay-Signature');

        // 1. Validate timestamp
        if (!$timestamp || abs(time() - (int)$timestamp) > 300) {
            Log::warning('CasaPay webhook: invalid timestamp');
            return response()->json(['error' => 'Invalid timestamp'], 401);
        }

        // 2. Verify signature
        $expectedSig = 'sha256=' . hash_hmac(
            'sha256',
            $timestamp . '.' . $rawBody,
            config('services.casapay.webhook_secret')
        );

        if (!hash_equals($expectedSig, $signature)) {
            Log::error('CasaPay webhook: invalid signature', ['ip' => $request->ip()]);
            return response()->json(['error' => 'Invalid signature'], 401);
        }

        // 3. Parse event
        $event = json_decode($rawBody, true);
        $sessionId = $event['session_id'];
        $eventType = $event['event'];

        // 4. Idempotency
        if (ProcessedWebhook::where('session_id', $sessionId)
                ->where('event_type', $eventType)->exists()) {
            return response()->json(['status' => 'already_processed']);
        }

        // 5. Handle event
        match ($eventType) {
            'gateway.session.completed' => $this->handleCompleted($event),
            'gateway.session.expired',
            'gateway.session.cancelled' => $this->handleTerminated($event),
            'gateway.payment.failed' => $this->handleFailed($event),
            default => Log::info("Unhandled CasaPay event: {$eventType}"),
        };

        // 6. Record
        ProcessedWebhook::create([
            'session_id' => $sessionId,
            'event_type' => $eventType,
            'processed_at' => now(),
        ]);

        return response()->json(['received' => true]);
    }

    private function handleCompleted(array $event): void
    {
        $sessionId = $event['session_id'];

        // CRITICAL: Verify via API
        $apiSession = Http::withToken(config('services.casapay.api_key'))
            ->get(config('services.casapay.api_url') . "/sessions/{$sessionId}")
            ->json();

        if (($apiSession['status'] ?? null) !== 'completed') {
            Log::warning("CasaPay API status mismatch", [
                'session_id' => $sessionId,
                'api_status' => $apiSession['status'] ?? 'unknown',
            ]);
            return;
        }

        $order = Order::where('gateway_session_id', $sessionId)->first();
        if (!$order) {
            Log::error("No order for CasaPay session: {$sessionId}");
            return;
        }

        $order->update([
            'gateway_status' => 'completed',
            'paid_at' => $event['payment']['paid_at'],
            'cash_amount' => $event['payment']['cash_amount'],
            'guaranteed_amount' => $event['payment']['guaranteed_amount'],
            'guarantee_id' => $event['guarantee']['guarantee_id'] ?? null,
        ]);

        // Downstream
        event(new PaymentCompleted($order));
    }
}
```

---

## Common Pitfalls

### ❌ Pitfall 1: Trusting the Return URL

**Problem:** Marking an order as paid when the customer lands on `success_url`.
**Attack:** Anyone can visit `yoursite.com/success?session=gwy_xxx` directly.
**Fix:** Only mark orders as paid after webhook verification + API confirmation.

### ❌ Pitfall 2: Not Verifying Signatures

**Problem:** Accepting webhooks without checking `X-CasaPay-Signature`.
**Attack:** Attacker sends fake webhook to your endpoint.
**Fix:** Always verify HMAC-SHA256 signature using constant-time comparison.

### ❌ Pitfall 3: Using Parsed Body for Signature

**Problem:** Verifying signature against `JSON.stringify(parsedBody)` instead of raw body.
**Issue:** JSON serialization may reorder keys, breaking signature.
**Fix:** Always use the raw request body bytes for signature computation.

### ❌ Pitfall 4: Non-Idempotent Webhook Handler

**Problem:** Creating duplicate records when the same webhook is received twice.
**Fix:** Track processed webhook `session_id + event_type` and skip duplicates.

### ❌ Pitfall 5: Slow Webhook Processing

**Problem:** Doing heavy work synchronously in the webhook handler (emails, PDFs, etc.).
**Fix:** Return `200 OK` immediately and process asynchronously (queue/background job).

### ❌ Pitfall 6: No Replay Attack Protection

**Problem:** Accepting webhooks with old timestamps.
**Fix:** Reject webhooks where `X-CasaPay-Timestamp` is more than 5 minutes old.

### ❌ Pitfall 7: Exposing API Keys in Frontend

**Problem:** Making API calls from JavaScript in the browser.
**Fix:** Always make CasaPay API calls from your backend only.

### ❌ Pitfall 8: Not Handling `guarantee` Data

**Problem:** Only looking at `payment.cash_amount` and missing the guarantee portion.
**Fix:** Always check both `payment.cash_amount` AND `payment.guaranteed_amount`. The `guarantee` object tells you if CasaPay is covering the deposit.

---

## Testing Guide

### Sandbox Testing

Use `sk_test_` keys with the sandbox API URL for all testing.

### Test Scenarios

| # | Scenario | How to Test | Expected |
|---|----------|-------------|----------|
| 1 | Full upfront payment | `deposit_mode: "deposit_upfront"` | Webhook: cash_amount = cover + first_payment |
| 2 | Full guarantee | `deposit_mode: "deposit_guaranteed"` | Webhook: guaranteed_amount = cover, cash_amount = first_payment |
| 3 | Customer choice → upfront | `deposit_mode: "choice"`, choose upfront | Same as #1 |
| 4 | Customer choice → guarantee | `deposit_mode: "choice"`, choose guarantee | Same as #2 |
| 5 | Session expiry | Create session, don't complete | `gateway.session.expired` webhook |
| 6 | Cancel session | Call cancel endpoint | `gateway.session.cancelled` webhook |
| 7 | Webhook retry | Return 500 from webhook endpoint | Webhook re-sent per retry policy |
| 8 | Invalid signature | Tamper with webhook body | Your handler returns 401 |
| 9 | Duplicate webhook | Replay same webhook | Handled idempotently |
| 10 | Return URL only | Visit success_url without webhook | Order stays "processing" |

### Webhook Testing Locally

Use a tunnel service (ngrok, localtunnel) to expose your local webhook endpoint:

```bash
# Start tunnel
ngrok http 3000

# Set webhook URL in CasaPay to:
# https://abc123.ngrok.io/webhooks/casapay
```

### Automated Test Template

```
test "webhook_signature_verification":
    valid_body = '{"event":"gateway.session.completed","session_id":"test"}'
    timestamp = str(int(time()))
    secret = "test_webhook_secret"
    
    // Valid signature
    valid_sig = "sha256=" + hmac_sha256(secret, timestamp + "." + valid_body).hex()
    assert verify_webhook(valid_body, timestamp, valid_sig, secret) == true
    
    // Invalid signature
    assert verify_webhook(valid_body, timestamp, "sha256=invalid", secret) == false
    
    // Tampered body
    tampered = '{"event":"gateway.session.completed","session_id":"hacked"}'
    assert verify_webhook(tampered, timestamp, valid_sig, secret) == false
    
    // Old timestamp
    old_timestamp = str(int(time()) - 600)
    old_sig = "sha256=" + hmac_sha256(secret, old_timestamp + "." + valid_body).hex()
    // Should be rejected by timestamp check (> 5 min old)

test "idempotent_webhook_processing":
    // Send same webhook twice
    result1 = handle_webhook(completed_event)
    assert result1.status == 200
    assert db.orders.find(session_id).status == "completed"
    
    result2 = handle_webhook(completed_event)  // Same event again
    assert result2.status == 200
    assert result2.body == {"status": "already_processed"}
    // No duplicate records created

test "success_url_does_not_fulfill_order":
    // Visit success URL directly without webhook
    GET /payment/success?session=gwy_test123
    assert order.status == "pending"  // NOT "completed"
    assert page.shows("Processing your payment...")
```

---

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│                  CasaPay Gateway Quick Ref                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  PRICING PREVIEW:                                             │
│    POST /api/v1/gateway/pricing-preview                       │
│    Auth: Bearer sk_live_xxx                                   │
│    Body: { cover_amount, first_payment_amount }               │
│                                                               │
│  CREATE SESSION (session-first):                              │
│    POST /api/v1/gateway/sessions                              │
│    Auth: Bearer sk_live_xxx                                   │
│    Body: { tenant, cover_amount, deposit_mode, ... }          │
│    → Returns: session_id, gateway_url                         │
│                                                               │
│  CREATE AGREEMENT (agreement-first, no payment):              │
│    POST /api/v1/gateway/agreements                            │
│    Auth: Bearer sk_live_xxx                                   │
│    Body: { tenant, agreement_type, cover_amount? }            │
│    → Returns: agreement_id, email_alias                       │
│                                                               │
│  CREATE INVOICE (follow-up):                                  │
│    POST /api/v1/gateway/agreements/{id}/invoice               │
│    Auth: Bearer sk_live_xxx                                   │
│    Body: { amount, success_url, cancel_url }                  │
│                                                               │
│  GET AGREEMENT:                                               │
│    GET /api/v1/gateway/agreements/{id}                        │
│    Auth: Bearer sk_live_xxx                                   │
│    → email_alias, coverage_amount, type, status, dates        │
│                                                               │
│  GET SESSION:                                                 │
│    GET /api/v1/gateway/sessions/{session_id}                  │
│    Auth: Bearer sk_live_xxx                                   │
│                                                               │
│  CANCEL SESSION:                                              │
│    POST /api/v1/gateway/sessions/{session_id}/cancel          │
│    Auth: Bearer sk_live_xxx                                   │
│                                                               │
│  DEPOSIT MODES:                                               │
│    deposit_upfront    → Tenant pays full deposit cash         │
│    deposit_guaranteed → CasaPay guarantees deposit            │
│    choice             → Tenant picks on checkout page         │
│                                                               │
│  WEBHOOK SIGNATURE:                                           │
│    HMAC-SHA256(secret, timestamp + "." + raw_body)            │
│    Header: X-CasaPay-Signature: sha256=<hex>                  │
│    Header: X-CasaPay-Timestamp: <unix_seconds>                │
│                                                               │
│  SECURITY RULES:                                              │
│    ✅ Verify webhook signatures (HMAC-SHA256)                 │
│    ✅ Double-check status via GET /sessions/{id}              │
│    ✅ Use constant-time comparison                            │
│    ✅ Reject timestamps > 5 min old                           │
│    ✅ Make webhook handler idempotent                         │
│    ✅ Process webhooks async (return 200 fast)                │
│    ❌ NEVER trust success_url alone                           │
│    ❌ NEVER expose API keys in frontend                       │
│    ❌ NEVER skip signature verification                       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Auto-Update (AI Agents)

This skill file includes a version check mechanism. AI agents should periodically verify they have the latest version.

**Current Version:** `1.4`
**Version Check URL:** `https://raw.githubusercontent.com/CasaPay/casapay-gateway-skill/main/VERSION`
**Full Skill URL:** `https://raw.githubusercontent.com/CasaPay/casapay-gateway-skill/main/README.md`

### For AI Agents: Self-Update Instructions

When starting a new CasaPay Gateway integration task, or if more than 7 days have passed since last check:

1. **Check for updates** by fetching the VERSION file:
   ```
   curl -s https://raw.githubusercontent.com/CasaPay/casapay-gateway-skill/main/VERSION
   ```
2. **Compare** the remote version with the version in this file header (`Version: 1.4`)
3. **If newer version exists**, download and replace this file:
   ```
   curl -s -o <path-to-this-file> https://raw.githubusercontent.com/CasaPay/casapay-gateway-skill/main/README.md
   ```
4. **Notify the developer** that the skill was updated

### For CI/CD: Automated Update Script

```bash
#!/bin/bash
# Check and update CasaPay Gateway skill file
SKILL_PATH=".cursor/rules/casapay-gateway.md"  # or wherever you store it
REMOTE_VERSION=$(curl -s https://raw.githubusercontent.com/CasaPay/casapay-gateway-skill/main/VERSION 2>/dev/null)
LOCAL_VERSION=$(grep -oP 'Version:\s*\K[\d.]+' "$SKILL_PATH" 2>/dev/null)

if [[ "$REMOTE_VERSION" != "$LOCAL_VERSION" && -n "$REMOTE_VERSION" ]]; then
    echo "🔄 CasaPay Gateway skill update available: $LOCAL_VERSION → $REMOTE_VERSION"
    curl -s -o "$SKILL_PATH" https://raw.githubusercontent.com/CasaPay/casapay-gateway-skill/main/README.md
    echo "✅ Updated to v$REMOTE_VERSION"
else
    echo "✅ CasaPay Gateway skill is up to date (v$LOCAL_VERSION)"
fi
```

---

## Support

For API keys, webhook configuration, or technical questions, visit [casapay.com](https://casapay.com) or reach out to your CasaPay integration contact.
