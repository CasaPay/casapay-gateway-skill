# CasaPay Gateway Skill — Changelog

All notable changes to the public skill + Postman collection are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
