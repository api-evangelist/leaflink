---
name: Reconcile inventory across facilities
description: Read LeafLink inventory measurements, compare against an external system, and issue inventory commands.
api: openapi/leaflink-api-openapi-original.yml
operations:
  - inventory_measurements_by_product_retrieve
  - inventory_measurements_by_facility_retrieve
  - inventory_commands_compare_inventory_create
  - inventory_commands_transfer_create
  - inventory_commands_hold_create
  - inventory_commands_unhold_create
  - inventory_commands_status_retrieve
  - facilities_list
  - batches_list
  - batches_create
generated: '2026-08-01'
method: generated
---

# Reconcile inventory across facilities

## Goal
Keep LeafLink inventory truthful against a warehouse or seed-to-sale system, and move stock safely.

LeafLink's inventory surface is **CQRS**: reads come from `inventory/measurements/*` queries, writes go
through `inventory/commands/*` and are **asynchronous**.

## Steps

1. **Enumerate facilities** with `facilities_list`. Inventory is facility-scoped; a reconciliation that
   ignores facility will produce wrong deltas for multi-site operators.
2. **Read current state.** Use `inventory_measurements_by_facility_retrieve` for a site-level view and
   `inventory_measurements_by_product_retrieve` for a SKU-level view. Finer-grained variants exist
   (by product+facility, by product+batch, by facility+batch) — pick the narrowest query that answers
   the question rather than pulling everything.
3. **Enumerate batches** with `batches_list` where lot-level accuracy matters; create missing lots with
   `batches_create`.
4. **Ask LeafLink to diff** with `inventory_commands_compare_inventory_create` rather than computing the
   comparison yourself when you can — it is the provider's own reconciliation primitive.
5. **Every command is async.** A command endpoint returns a command handle; poll
   `inventory_commands_status_retrieve` until it settles. Do not assume success from the POST response,
   and do not re-issue the command while the first one is still in flight — there is no idempotency key,
   so a duplicated transfer moves stock twice.
6. **Move stock** with `inventory_commands_transfer_create`. **Quarantine** with
   `inventory_commands_hold_create` and release with `inventory_commands_unhold_create`.
7. **Re-read** the measurement query from step 2 to confirm the command actually landed.

## Cautions
- Inventory commands are consequential physical-world writes. Gate `transfer`, `hold`, `destroy`,
  `archive` and `fulfillment` behind human approval — see `agentic-access/leaflink-agentic-access.yml`.
- The `inventory-v2/debug/*` endpoints are diagnostic. Never write logic against them.

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
