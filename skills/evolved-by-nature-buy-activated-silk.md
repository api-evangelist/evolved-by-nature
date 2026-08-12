---
name: buy-activated-silk
description: >-
  Search, select and purchase Evolved By Nature Activated Silk product from either storefront using
  the live UCP/MCP commerce surface, with buyer approval at payment.
api: evolved-by-nature-skincare-commerce
generated: '2026-08-12'
method: generated
source: >-
  Grounded in the tool names and required parameters returned by POST tools/list at
  https://skincare.evolvedbynature.com/api/ucp/mcp on 2026-08-12
  (mcp/evolved-by-nature-skincare-ucp-tools.json). No tool, parameter or endpoint below is invented.
operations:
- search_catalog
- get_product
- lookup_catalog
- create_cart
- update_cart
- get_cart
- create_checkout
- update_checkout
- complete_checkout
- get_order
---

# Buy Activated Silk product

Two storefronts, same tool surface, different catalogs:

| Store | Endpoint | Sells |
|---|---|---|
| Skincare | `https://skincare.evolvedbynature.com/api/ucp/mcp` | Finished consumer skincare |
| Bioactives | `https://bioactives.evolvedbynature.com/api/ucp/mcp` | Activated Silk ingredient grades |

Transport is MCP streamable-http over JSON-RPC 2.0. Send
`Accept: application/json, text/event-stream`. No credentials are needed to discover or to browse.

## Before anything else

**Every UCP tool requires `meta.ucp-agent.profile`** — a resolvable URI identifying your agent. The
server resolves the agent profile *before* it validates tool arguments, so a missing profile masks
every other error:

```
HTTP 422
{"jsonrpc":"2.0","error":{"code":-32001,"message":"UCP discovery failed",
 "data":{"code":"invalid_profile_url","content":"Unable to fetch agent profile: Missing profile uri"}}}
```

Confirm capabilities first at `GET /.well-known/ucp`. Current version is `2026-04-08`; `2026-01-23`
is still served.

## Steps

1. **Search** — `search_catalog` with `meta` and `catalog`. Pass `catalog.context.address_country`
   and `catalog.context.currency` so pricing and availability are correct. At least one of
   `catalog.query` or `catalog.filters` must be present. Results are paginated; follow
   `pagination.cursor` for more.
2. **Inspect** — `get_product` (single, by identifier) or `lookup_catalog` (several at once). Both
   require `meta` and `catalog`.
3. **Cart** — `create_cart` with `meta` and `cart`; then `update_cart` (`meta`, `cart`, `id`) to
   adjust line items. `get_cart` (`meta`, `id`) reads it back. `cancel_cart` abandons it.
4. **Checkout** — `create_checkout` (`meta`, `checkout`), then `update_checkout`
   (`meta`, `checkout`, `id`) to set the shipping address and method. `get_checkout` (`meta`, `id`)
   returns line items, totals, discounts and taxes.
5. **Complete** — `complete_checkout` (`meta`, `id`, `checkout`). **Do not call this without an
   explicit, contemporaneous human approval of the payment.** The store's own `robots.txt` and
   `agents.md` both prohibit autonomous checkout completion.
6. **Confirm** — `get_order` (`meta`, `id`).

## Rules that will bite you

- **Money is minor units.** `{"amount": 2500, "currency": "USD"}` is $25.00. Divide by 100 for
  two-decimal currencies before quoting a buyer; JPY and other zero-decimal currencies are already
  whole units.
- **Identifiers are Shopify GIDs**, e.g. `gid://shopify/Checkout/abc123`.
- **No idempotency.** Nothing on this surface exposes an idempotency key or a retry-safety contract.
  A retried `create_cart` or `create_checkout` creates another one. Track ids yourself.
- **Rate limits are per-IP and unpublished.** No numbers are documented and no `RateLimit-*` headers
  were observed. Back off on `429`.
- **Fulfillment is single-destination.** `/.well-known/ucp` declares
  `allows_multi_destination.shipping: false` and only the `[shipping]` method combination.

## Read-only alternative

If you only need catalog data and not a transaction, `agents.md` points at unauthenticated JSON:
`GET /products.json`, `GET /products/{handle}.json`, `GET /collections/{handle}/products.json`,
`GET /search?q={query}&type=product`. `robots.txt` disallows `/cart.js` and
`/recommendations/products` — use the MCP surface for those instead.
