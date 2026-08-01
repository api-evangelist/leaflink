---
name: Sync a brand product catalog
description: Read, create and update a LeafLink brand product catalog, including images, categories and strains.
api: openapi/leaflink-api-openapi-original.yml
operations:
  - products_list
  - products_retrieve
  - products_create
  - products_partial_update
  - product_images_list
  - product_images_create
  - product_categories_list
  - product_subcategories_list
  - product_lines_list
  - strains_list
  - product_listing_states_list
generated: '2026-08-01'
method: generated
---

# Sync a brand product catalog

## Goal
Keep a brand's LeafLink product catalog in sync with an external system of record (ERP or seed-to-sale).

## Steps

1. **Resolve the taxonomy first.** Call `product_categories_list` and `product_subcategories_list`, plus
   `product_lines_list` and `strains_list` if the catalog uses them. LeafLink categories are a controlled
   vocabulary — map external categories onto the returned IDs before writing anything. Also read
   `product_listing_states_list` to learn which listing states are valid.
2. **Read what already exists.** Call `products_list` and page through with `page`/`page_size` until
   `next` is null. Filter with `search` where you can. Build an index keyed on your own external ID —
   LeafLink stores it on the product's `external_ids` field, which is the only durable join key you get.
3. **Diff, do not blind-write.** For each external product, decide create vs update from the index built
   in step 2. Never `POST` speculatively: there is no idempotency key, so a retried create yields a
   duplicate listing that a human has to clean up.
4. **Create** with `products_create`, setting `external_ids` in the same request so step 2 can match it
   on the next run.
5. **Update** with `products_partial_update` (PATCH). Send only changed fields. Read the current record
   with `products_retrieve` first when you need to compute a delta.
6. **Attach images** with `product_images_create`, and reconcile against `product_images_list` so you do
   not re-upload the same asset every run.
7. **Verify** with `products_retrieve` on a sample of writes before declaring the sync clean.

## Cautions
- `unit_of_measure` no longer accepts `"Case"` (removed in Marketplace V2 2.27.0) — use
  `sell_in_unit_of_measure` together with `unit_multiplier`.
- Keep writes under 8 requests/second. Batch reads with `page_size=500` to spend your budget on writes.

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
