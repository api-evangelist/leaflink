---
name: Reconcile payments and invoices
description: Read LeafLink sales invoices, record payments, and inspect payment contracts and tax schedules.
api: openapi/leaflink-api-openapi-original.yml
operations:
  - sales_invoices_list
  - sales_invoices_retrieve
  - payments_list
  - payments_create
  - payments_retrieve
  - invoice_recorded_payments_create
  - payment_contracts_list
  - taxes_tax_schedules_list
generated: '2026-08-01'
method: generated
---

# Reconcile payments and invoices

## Goal
Reconcile LeafLink Financial invoices and payments against an accounting ledger.

## Steps

1. **Pull invoices** with `sales_invoices_list`, paging until `next` is null. Read one with
   `sales_invoices_retrieve`.
2. **Pull payments** with `payments_list` and `payments_retrieve`.
3. **Understand the terms** with `payment_contracts_list` — net-terms agreements determine what is
   actually due and when. Read `taxes_tax_schedules_list` when you need to explain a tax line.
4. **Match** invoices to payments on your ledger's reference, carried in `external_ids`.
5. **Record an off-platform payment** with `invoice_recorded_payments_create`, or create a payment with
   `payments_create`.
6. **Verify** by re-reading the invoice with `sales_invoices_retrieve` and confirming the balance moved.

## Cautions
- **Money movement with no idempotency key.** `payments_create` and `invoice_recorded_payments_create`
  will happily double-post on retry. Before any retry, re-query `payments_list` filtered on your
  reference and only re-issue when the read proves the payment is absent. On an ambiguous timeout
  (`408`, dropped connection), always read before you write again.
- Require human approval for any payment creation. Never auto-retry a failed payment in a loop.
- LeafLink publishes no decline-code reference; a payment failure surfaces as a generic error, so
  escalate rather than trying to classify it.

## LeafLink ground rules (apply to every step)

- **Base URL** — `https://api.leaflink.com` for the current API. The legacy Marketplace V2 API is
  `https://app.leaflink.com/api/v2/`. Test against `https://staging-api.leaflink.com` (current) or
  `https://www.sandbox.leaflink.com/api/v2/` (legacy).
- **Auth** — send `Authorization: Bearer <JWT>` on the current API. The legacy V2 API uses
  `Authorization: App <api_key>` (one space, single-company scope) or a User Token (all companies the
  holder can access). Unauthenticated calls return `401`.
- **Version** — send `LeafLink-Version: 2022-10-31`. Versions are dates; omitting the header silently
  binds you to whatever is current, so always pin it.
- **Trailing slash** — current-API paths must NOT end in `/` (a trailing slash returns `404`). Legacy V2
  paths MUST end in `/` (omitting it returns `400`). This is the single most common integration bug.
- **Paging** — `page` + `page_size` (max 500, default 50) + `ordering`; read `count`/`next` from the
  envelope and follow `next` until null. Legacy V2 uses `limit`/`offset`.
- **Rate limits** — 300 requests/minute and 8 requests/second per API key owner. Watch
  `RateLimit-Remaining` and back off on `429`; keep sustained throughput under 8 rps.
- **NO IDEMPOTENCY.** LeafLink documents no idempotency key and neither OpenAPI declares one. A retried
  `POST` will create a duplicate. Before retrying any create, re-query by `external_ids` (or the natural
  key) and only create when the read confirms absence.
- **Errors** are plain `application/json`, not RFC 9457. Expect `400`, `401`, `404`, `408`, `409`, `429`,
  `500`. See `errors/leaflink-problem-types.yml`.
- **No CORS** — this API cannot be called from a browser; run server-side.
- Everything is `snake_case`, request and response bodies are `application/json` only.
