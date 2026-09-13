---
name: agilitassports-browse-the-catalog
description: >-
  Search the Agilitas catalog (Lotto footwear and apparel, one8), resolve product and variant detail,
  and quote prices to a buyer without getting the currency wrong.
api: Agilitas Commerce MCP API
endpoint: https://agilitas.com/api/ucp/mcp
transport: mcp
auth: none
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-09-12'
method: generated
source: mcp/agilitassports-ucp-mcp-tools.json (live tools/list, 2026-09-12)
---

# Browse the Agilitas catalog

Read-only. Nothing in this skill creates a cart, moves money, or changes anything at the merchant.

## Before you start

Both published endpoints work, which is unusual and worth knowing:

- `https://agilitas.com/api/ucp/mcp` — named in `/llms.txt`, `/agents.md` and `/robots.txt`. **HTTP 200.**
- `https://aq0zmc-sj.myshopify.com/api/ucp/mcp` — named in `ucp.services["dev.ucp.shopping"][0].endpoint`
  of `https://agilitas.com/.well-known/ucp`. **Also HTTP 200.**

Prefer the discovery document, since that is the one the protocol tells you to read. The endpoint is
POST-only; a `GET` returns 404 and that is not an outage.

No credential is needed. But **every** tool call requires the agent-identity block:

```json
"meta": { "ucp-agent": { "profile": "<your agent profile URI>" } }
```

Omit it and the call fails before dispatch with JSON-RPC `-32001 / invalid_profile_url`. That check
runs first, so a missing profile masks every other error you might be trying to diagnose.

## 1. Search

Call `search_catalog` with `catalog.query` and/or `catalog.filters`.

```json
{
  "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": {
    "name": "search_catalog",
    "arguments": {
      "meta": { "ucp-agent": { "profile": "https://example.com/agent" } },
      "catalog": {
        "query": "running shoes",
        "filters": { "available": true, "price": { "min": 0, "max": 500000 } },
        "context": { "address_country": "IN", "currency": "INR" },
        "pagination": { "limit": 10 }
      }
    }
  }
}
```

- `filters.price.min` / `.max` are **minor units** — `500000` is ₹5,000.00.
- `filters.available` defaults to `true` (sale-ready items only).
- `pagination.limit` defaults to `10`, minimum `1`. Page with `pagination.cursor` from the response.
- `filters.categories[]` combines with OR logic.

## 2. Resolve detail

- `lookup_catalog` — resolve a batch of product or variant GIDs in one call.
- `get_product` — full detail for one `gid://shopify/Product/{id}`, including variants, exact pricing
  and real-time availability. Pass `catalog.selected` to pin option choices.

## 3. Quote the price correctly

Every money value is an integer in ISO 4217 **minor units** paired with a currency code:

```json
{ "amount": 2500, "currency": "USD" }   // $25.00
```

Divide by 100 for two-decimal currencies before you say a number out loud. Zero-decimal currencies
such as JPY are already whole units. This store is India-facing — expect INR.

## Localization

Pass `catalog.context` — `address_country` (ISO 3166-1 alpha-2), `address_region`, `postal_code`,
`language` (BCP 47), `currency` (ISO 4217), `intent`. These are hints: authoritative data such as a
real shipping address supersedes them, and unsupported hints are ignored without error.

`catalog.signals` carries `dev.ucp.buyer_ip` and `dev.ucp.user_agent`. Sending them forwards your
buyer's IP and user agent to the merchant — get consent, or leave them out.

## What you will not find

The catalog is small. `https://agilitas.com/products.json` returned **4 products** on 2026-09-12, all
Lotto footwear. If a search comes back thin, that is the shop, not your query. There is no
authenticated customer surface, no subscription product and no inventory-location data in this
contract.

If the buyer asks a policy question (returns, shipping, hours), use the **storefront** MCP server at
`https://agilitas.com/api/mcp`, whose single tool `search_shop_policies_and_faqs` answers exactly
that.
