# CasaPay Gateway Skill — Changelog

All notable changes to the public skill + Postman collection are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.10] — 2026-05-15

### Added
- **Soft-error webhooks (`gateway.payment.failed`).** The event existed in the
  enum but was never dispatched. It now fires whenever a payment attempt fails
  in a way that lets the tenant retry on the same gateway URL — bank declined,
  insufficient funds, customer cancelled at the bank, 3DS failed, payment
  timeout. The session stays in `pending` + `current_step=payment` so the
  tenant can try a different bank or card; the operator gets notified via
  webhook so they can surface the failure in their UI / send a follow-up
  reminder.
  - New `failure` block on the payload: `reason` (stable enum), `code`,
    `message`, `retry_count`, `can_retry`, `payment_method`, plus provider
    extras (`provider`, `provider_state`).
  - Reason enum: `payment_failed`, `payment_timeout`, `bank_declined`,
    `insufficient_funds`, `customer_cancelled`, `3ds_failed`.
  - Wired into the real EveryPay paths: `handleEveryPayWebhook` and
    `handleEveryPayReturn` map `failed`/`voided`/`abandoned` payment states to
    the matching reason and dispatch the webhook.
  - Outbound `gateway.payment.failed` webhooks are recorded in
    `gateway_request_logs` (alongside inbound API traffic), so operators can
    inspect every soft failure for a session via
    `GET /sessions/{id}/logs?direction=outbound&event_type=gateway.payment.failed`.
- **Simulator parity for soft errors.** Test-mode sessions can now replicate
  every production soft-error end-to-end via the simulator endpoint:
  - New scenarios: `payment_bank_declined`, `payment_insufficient_funds`,
    `payment_customer_cancelled`, `payment_3ds_failed` (in addition to the
    existing `payment_failed` and `payment_timeout`).
  - Each scenario dispatches the same `gateway.payment.failed` webhook with the
    matching `failure.reason` so operators can verify their handler logic
    against the same payload shape they'll see in production.
  - `POST /api/v1/gateway/public/sessions/{id}/simulator` validator updated.

### Changed
- Skill doc: webhook section reorganised — `gateway.payment.failed` now has its
  own "Soft Payment Errors" subsection with the failure-block payload, the
  reason enum table, and a curl example for replicating each scenario via the
  simulator.
- Skill doc: `gateway.payment.failed` row in the Event Types table is now
  flagged as **soft** (does not terminate the session) with a link to the
  detailed subsection.
- Auto-update version bumped to `1.10`.

### Tests
- New: `tests/Feature/GatewayPaymentFailedWebhookTest.php` — covers the helper,
  payload shape, all simulator scenarios (parameterised dataset), the public
  simulator endpoint, EveryPay webhook → reason mapping, and persistence in
  `gateway_request_logs`.

## [1.9] — 2026-05-13

### Added
- **`due_date` field documented** for `POST /agreements/{id}/invoice`. The API
  has accepted this field since inception but it was previously undocumented in
  the skill and Postman collection, so integrators couldn't align payout
  timing with their billing cycle.
  - Skill doc now includes the full field reference table (amount, success_url,
    cancel_url, description, document_url, `due_date`, metadata) on the
    invoice endpoint, with notes that `due_date` drives payout timing
    (`ontime` = due date, `cover` = due date + 30-day grace) and defaults to
    today if omitted.
  - Quick Reference Card updated to list `due_date?` on the invoice body.
  - Postman collection "Create Rent Invoice" request now includes
    `"due_date": "2026-03-01"` in the example body and documents it in the
    request description.


### Changed
- Invoice Document URL endpoint section: added clear guidance that the signed
  URL **expires** and should be **fetched on-demand** (when the user clicks
  "View invoice") rather than cached or stored in your database. Includes a
  do/don't usage-pattern example and a recommendation to use short expiry
  windows (10–60 min) when redirecting the browser immediately.
- Quick Reference Card note updated to flag the same expiry caveat.

### Fixed (backend, not doc)
- `createInvoiceSession()` now attaches a `Document` record to the created
  Invoice when `document_url` is provided — previously only stored the PDF
  in S3 without the DB link, causing `GET /invoices/{id}/document` to return
  `DOCUMENT_NOT_FOUND` for any invoice created via `POST /agreements/{id}/invoice`.
- `getInvoiceDocument()` fallback now generates a proper signed URL (via the
  app-level `documents.view` route) for legacy invoices that pre-date the
  Document record, instead of returning the raw public S3 URL (which failed
  with AccessDenied when the bucket wasn't publicly readable).

## [1.7] — 2026-05-12

### Added
- New **Monitoring & Observability** top-level section with concrete metrics,
  alert thresholds, dashboard ideas, and retention guidance.
- CHANGELOG file published alongside the skill and Postman collection.

## [1.6] — 2026-05-12

### Added
- **Session Logs endpoint** (`GET /sessions/{id}/logs`) with direction /
  success / pagination filters for debugging webhook delivery.
- **List Agreement Invoices** section (`GET /agreements/{id}/invoices`) with
  payout status filters.
- **Invoice Document URL** section (`GET /invoices/{id}/document`) returning a
  temporary signed URL for the invoice PDF.
- Quick Reference Card updated with the three new endpoints.
- Postman: `2.4 List Session Logs`, `3.4 List Agreement Invoices`,
  `3.5 Get Invoice Document URL`.

## [1.5] — 2026-04-17

### Added
- `agreement_type: cover` (reactive guarantee, payout delayed 30 days).
- Agreement-first flow notes and examples.

## [1.4.1] — 2026-04-16

### Fixed
- Minor corrections to Node.js and Python webhook examples.

## [1.4] — 2026-04-16

### Added
- Auto-update instructions for AI agents (VERSION file check).
- CI/CD update script example.

## [1.3.2] — 2026-04-16

### Fixed
- Clarified `deposit_mode: choice` behaviour and webhook `resolved_mode` field.

## [1.3.1] — 2026-04-16

### Changed
- Tightened security rules around raw-body signature verification.

## [1.3] — 2026-04-16

### Added
- Pricing Preview endpoint.
- Cover/first payment split model with side-by-side deposit options.

## [1.1.1] — 2026-04-07

### Fixed
- Corrected curl examples and `tenant.phone` regex description.

## [1.1] — 2026-04-07

### Added
- Webhook retry policy table.
- Polling fallback strategy (Option B).

## [1.0] — 2026-04-07

### Added
- Initial public release of the CasaPay Gateway Integration Skill.
- Full API reference (pricing, sessions, agreements).
- Webhook system + HMAC-SHA256 signature verification.
- Code samples for Node.js, Python, PHP/Laravel.
- Security rules, pitfalls, and testing guide.
