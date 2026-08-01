---
name: Dispatch a delivery and manage transportation
description: Create LeafLink deliveries and manage transportation orders, drivers, vehicles and distribution zones.
api: openapi/leaflink-api-openapi-original.yml
operations:
  - deliveries_list
  - deliveries_create
  - deliveries_retrieve
  - transportation_orders_list
  - transportation_orders_create
  - transportation_drivers_list
  - transportation_vehicles_list
  - transportation_partners_list
  - distribution_zones_list
generated: '2026-08-01'
method: generated
---

# Dispatch a delivery and manage transportation

## Goal
Turn a fulfilled wholesale order into a compliant cannabis delivery with an assigned driver and vehicle.

## Steps

1. **Check serviceability** with `distribution_zones_list` — confirm the destination falls inside a zone
   you actually deliver to before creating anything.
2. **Resolve carriers** with `transportation_partners_list` when using a third-party carrier.
3. **Resolve the crew** with `transportation_drivers_list` and `transportation_vehicles_list`. Cannabis
   transport is licensed: the driver and vehicle records exist because regulators require them. Do not
   dispatch against a driver or vehicle you could not read back.
4. **Check for an existing dispatch** with `transportation_orders_list` / `deliveries_list` filtered on
   the order reference. There is no idempotency key — a duplicate transport order means a real second
   truck run.
5. **Create the transport order** with `transportation_orders_create`, then the delivery with
   `deliveries_create`.
6. **Confirm** with `deliveries_retrieve`.
7. **Track** by polling `deliveries_list` on `modified__gte`, or subscribe to order webhooks.

## Cautions
- Dispatch is a physical-world, safety- and compliance-relevant action. Require human confirmation before
  `deliveries_create` and `transportation_orders_create`.
- Driver, vehicle and manifest data is regulated personal/licence data — do not log it into
  general-purpose stores.

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
