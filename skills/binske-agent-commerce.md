---
name: binske-agent-commerce
description: >-
  Browse the binske cannabis storefront and assemble a cart and checkout on a
  buyer's behalf over the store's UCP/MCP endpoint, stopping short of payment —
  which requires explicit, contemporaneous human approval.
generated: '2026-08-07'
method: generated
source:
  - https://shopbinske.com/agents.md
  - https://shopbinske.com/api/ucp/mcp (tools/list, 200, 2026-08-07)
api: binske Storefront Commerce API (UCP / MCP)
endpoint: https://shopbinske.com/api/ucp/mcp
protocol: mcp
transport: streamable-http
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
---

# Shopping the binske storefront as an agent

Every tool name and every field named below was read from a live `tools/list`
response on 2026-08-07. Nothing here is invented. The surface is Shopify's
implementation of the Universal Commerce Protocol, served from binske's own
storefront host.

## Before you call anything

1. `GET https://shopbinske.com/.well-known/ucp` and confirm the store still
   advertises `dev.ucp.shopping` and a protocol version you support
   (`2026-04-08` is current; `2026-01-23` is also served).
2. Every one of the 13 tools requires a `meta` object, and `meta` requires
   `ucp-agent.profile` — a dereferenceable URI identifying you. There is no
   anonymous, unattributed mode. Set it on every call.
3. Set `context.address_country` and `context.currency` so pricing and
   availability come back correct.
4. This is a regulated cannabis storefront. Age and jurisdiction restrictions
   apply, and binske publishes state-specific consumer warnings at
   <https://binske.com/compliance/> for CO, FL, MI, NJ, NY and WA. Do not route
   a purchase to a jurisdiction the buyer is not in.

## Find products

- `search_catalog` — natural-language query. Takes `catalog.query`,
  `catalog.filters`, and cursor pagination at `catalog.pagination.cursor` /
  `catalog.pagination.limit` (default 10, minimum 1). Page with the cursor; do
  not re-issue the same query with a larger limit.
- `lookup_catalog` — resolve several known product IDs in one call via
  `catalog.ids`. Prefer this over N single lookups.
- `get_product` — full detail for one `catalog.id`, with the relevant variants
  and exact pricing. Pass `catalog.selected` to pin variant options.

## Build the cart

- `create_cart` with `cart.line_items`. You may also set `cart.buyer` (email,
  phone), `cart.fulfillment`, and `cart.attribution`.
- `update_cart` requires both `cart` and `id`.
- `get_cart` / `cancel_cart` take `id`.
- Only apply `cart.discounts` if the buyer actually mentioned a code. The schema
  says so explicitly — do not go hunting for promo codes on your own.

## Move to checkout

- `create_checkout` accepts either `checkout.line_items` directly or
  `checkout.cart_id` to convert an existing cart. Converting is preferred: it
  keeps one source of truth for the line items.
- `update_checkout` (requires `checkout` and `id`) is where shipping address and
  method go, under `checkout.fulfillment`. This store allows a single shipping
  destination per checkout — no multi-destination splits.
- `get_checkout` (requires `id`) re-reads line items, totals, discounts and
  taxes. Read it back after every mutation rather than assuming your local
  totals are still right.
- Checkout IDs are Shopify global IDs, shaped `gid://shopify/Checkout/abc123`.

## Stop at payment

`complete_checkout` is the only money-moving tool, and it is the only one you
must not call on your own initiative.

- The store's own instructions: *"Checkout requires human approval. Agents must
  not complete payment without explicit buyer consent."* The robots.txt is
  blunter: *"Checkouts are for humans."*
- If you cannot obtain contemporaneous buyer approval at the moment of payment,
  do not complete the checkout — route the purchase through the Shop skill at
  <https://shop.app/SKILL.md>, which carries the buyer-approval step for you.
- When you do call it, pass `meta.idempotency-key`. It is the single idempotency
  key on this entire surface. Reuse the same key on any retry of the same
  intended purchase; never generate a fresh one for a retry.
- Payment instruments go under `checkout.payment.instruments[]`, each requiring
  `id`, `handler_id` and `type` (`card` or `token`). The store advertises three
  payment handlers: `dev.shopify.card`, `dev.shopify.shop_pay`, and
  `com.google.pay`.

## After the order

- `get_order` with `id` returns order detail.
- `cancel_checkout` / `cancel_cart` clean up abandoned state. Use them rather
  than leaving orphaned checkouts.

## Failure handling

- Errors arrive in the JSON-RPC 2.0 `error` member —
  `{"jsonrpc":"2.0","id":<id>,"error":{"code":<int>,"message":<string>}}`. There
  is no RFC 9457 problem document and no published error-code registry for this
  surface, so surface the raw `message` to the buyer rather than mapping it.
- The endpoint is rate-limited per IP. Back off on `429`; no limit numbers are
  published, so back off exponentially rather than tuning to a quota.
- If a `complete_checkout` call times out, do **not** retry with a new key —
  retry with the same `meta.idempotency-key`, or read the checkout back with
  `get_checkout` first.

## What this surface does not give you

binske runs no developer program. There is no OpenAPI, no SDK, no CLI, no
sandbox, no test mode, no webhooks or event stream, no changelog, no status
page, and no security.txt. Anything you need beyond catalog, cart, checkout and
order retrieval does not exist here.
