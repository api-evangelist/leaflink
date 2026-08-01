---
name: Process orders received
description: List, read, transition and reconcile wholesale orders received on the LeafLink Marketplace V2 API.
api: openapi/leaflink-marketplace-v2-openapi-original.yml
operations:
  - orders-received_list
  - orders-received_read
  - orders-received_create
  - orders-received_partial_update
  - orders-received_transition_create
  - orders-received-line-items_list
  - line-items_create
  - order-payments_list
  - order-event-logs_list
generated: '2026-08-01'
method: generated
---

# Process orders received

## Goal
Pull new and changed wholesale orders off LeafLink, push them into a fulfilment system, and drive each
order through its LeafLink status transitions.

> Orders live on the **legacy Marketplace V2 API** (`https://app.leaflink.com/api/v2/`). Every path here
> MUST end in a trailing slash, and auth is `Authorization: App <api_key>`.

## Steps

1. **Poll for changes, not everything.** Call `orders-received_list` filtered on `modified__gte` with the
   timestamp of your last successful run. Page with `limit`/`offset`. This is far cheaper than a full
   scan and keeps you inside the rate limit.
2. **Expand what you need in one call.** `orders-received_list` supports `include_children` (including
   `line-items`) and `fields_include`/`fields_exclude`. Pull line items inline rather than making an
   N+1 round trip; fall back to `orders-received-line-items_list` only for a single deep read.
3. **Read one order** with `orders-received_read` — note it is keyed on the order `number`, not an `id`.
4. **Push into fulfilment** using the order's `external_ids` field to carry your own reference back.
5. **Transition the order** with `orders-received_transition_create`, which takes the order `number` and
   an `action`. Drive the order forward through its real status path (accept, fulfil, complete) rather
   than PATCHing a status field.
6. **Amend** with `orders-received_partial_update` for things like `payment_term` and `payment_due_date`,
   which are PATCH-updatable. Add line items with `line-items_create`.
7. **Reconcile payment** with `order-payments_list` for the order.
8. **Audit** with `order-event-logs_list` when you need to explain what changed and when.

## Cautions
- `orders-received_create` exists (added in 2.39.0) but has **no idempotency key**. Only call it after a
  `orders-received_list` filtered on your `external_ids` confirms the order is absent.
- Prefer the webhook subscription "NEW & CHANGED ORDERS" over tight polling — see
  `asyncapi/leaflink-webhooks.yml`. Verify the `LL-Signature` HMAC-SHA256 header on every delivery.

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
