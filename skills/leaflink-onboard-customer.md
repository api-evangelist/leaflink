---
name: Onboard and verify a wholesale customer
description: Create a LeafLink customer, attach contacts, and verify cannabis licenses and status.
api: openapi/leaflink-api-openapi-original.yml
operations:
  - customers_list
  - customers_retrieve
  - customers_create
  - customers_partial_update
  - customer_statuses_list
  - customer_groups_list
  - compliance_licenses_list
  - compliance_license_types_list
  - contacts_list
generated: '2026-08-01'
method: generated
---

# Onboard and verify a wholesale customer

## Goal
Bring a new wholesale buyer onto a brand's LeafLink account with the compliance record LeafLink requires.

## Steps

1. **Check for an existing record first.** Call `customers_list` with `search` on the company name, and
   match on `external_ids` if you have one. Cannabis operators frequently appear under trading names —
   a duplicate customer splits order history and there is no idempotency key to protect you.
2. **Resolve the vocabularies.** Call `customer_statuses_list` for valid pipeline statuses and
   `customer_groups_list` for the groups you can assign. Call `compliance_license_types_list` to learn
   which license types are valid — they are state-scoped and carry a `state_abbr`.
3. **Create the customer** with `customers_create`. Set `external_ids` in the same request.
4. **Set addresses** — the customer object carries distinct `corporate_address` and `delivery_address`
   objects. Populate both; logistics depends on `delivery_address`.
5. **Verify licensing** with `compliance_licenses_list` scoped to the customer. A buyer without a valid,
   in-state license cannot legally transact. Treat a missing or expired license as a hard stop and
   escalate to a human — do not proceed to ordering.
6. **Attach people** with `contacts_list` to review, and keep the customer's `ein` in mind: it is
   inherited read-only from the buyer company's federal EIN (Marketplace V2 2.36.0), so do not try to
   set it.
7. **Advance the relationship** with `customers_partial_update` to move status as the account progresses.
8. **Confirm** with `customers_retrieve`.

## Cautions
- This flow touches regulated licence data. Never fabricate or infer a license number, and never
  auto-approve a customer whose license check failed — hand off to a human.

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
