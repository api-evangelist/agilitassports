---
name: agilitassports-build-an-order
description: >-
  Assemble an Agilitas cart, set the buyer and destination, convert it to a priced checkout, and back
  out cleanly if the buyer changes their mind. Stops short of payment.
api: Agilitas Commerce MCP API
endpoint: https://agilitas.com/api/ucp/mcp
transport: mcp
auth: none
operations:
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - cancel_checkout
generated: '2026-09-12'
method: generated
source: mcp/agilitassports-ucp-mcp-tools.json (live tools/list, 2026-09-12)
---

# Build an Agilitas order

This skill writes. It creates real carts and real checkouts against a live production store. It does
**not** complete payment — that is `agilitassports-complete-purchase-safely`.

Every call needs `meta["ucp-agent"].profile`. There is no sandbox, no test mode and no test payment
token, so there is nothing to rehearse against. Everything you create here is real but reversible.

## 1. Create the cart

```json
{
  "meta": { "ucp-agent": { "profile": "https://example.com/agent" } },
  "cart": {
    "line_items": [ { "item": { "id": "gid://shopify/ProductVariant/..." }, "quantity": 1 } ],
    "buyer": { "email": "buyer@example.com" },
    "context": { "address_country": "IN", "currency": "INR" }
  }
}
```

`line_items[].item.id` is a **Product Variant** ID, not a product ID. Resolve it with `get_product`
first.

## 2. Adjust

`update_cart` changes line items (addressed by line-item `id`), `buyer`, `context`, fulfillment, or
`discounts.codes[]`.

- `discounts.codes[]` is **full replacement**: a new array replaces the previous codes, and an empty
  array clears them. That makes a discount update self-reversing.
- Only ask about a discount code if the buyer mentions having one.

`get_cart` re-reads the cart from its `gid://shopify/Cart/{id}`.

## 3. Convert to a checkout

`create_checkout` takes either fresh `line_items` or an existing cart, plus `buyer`, fulfillment
methods and destinations, and `discounts.codes[]`. The response carries line items, totals,
discounts and taxes — this is where the buyer finally sees a real, shippable price.

`update_checkout` revises any of it. `get_checkout` re-reads it.

## 4. Backing out

Both write surfaces have an explicit reversal, and this is the whole window in which an agent can
undo its own work:

| You created | Undo with | Until |
|---|---|---|
| a cart | `cancel_cart` | any time before payment |
| a checkout | `cancel_checkout` | any time before `complete_checkout` |

After `complete_checkout` succeeds there is **no** reversal tool. Nothing in this API refunds,
voids or cancels an order.

## Retry discipline

Only `complete_checkout` accepts an idempotency key. `create_cart`, `create_checkout`, `update_cart`
and `update_checkout` are **not** idempotent — a blind retry after a timeout creates a second object.
If you must retry, retry, then reconcile: look for the duplicate and `cancel_cart` /
`cancel_checkout` the orphan.

## Attribution

`attribution` accepts `referring_domain`, `click_id_tag`, `click_id_value` and the `utm_*` fields on
cart and checkout writes. The merchant expects agents to declare where the buyer came from. Fill it
honestly or leave it empty; do not invent a source.

## Errors

Application errors come back inside an **HTTP 200** as a JSON-RPC `error` object. Branch on the
presence of `result` vs `error`, never on the status code. See
`errors/agilitassports-problem-types.yml`.
