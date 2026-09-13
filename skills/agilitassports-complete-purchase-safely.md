---
name: agilitassports-complete-purchase-safely
description: >-
  Complete an Agilitas checkout with contemporaneous buyer approval and an idempotency key, confirm
  the order, and tell the buyer accurately what can still be undone.
api: Agilitas Commerce MCP API
endpoint: https://agilitas.com/api/ucp/mcp
transport: mcp
auth: none (buyer-supplied payment instrument per call)
operations:
  - update_checkout
  - complete_checkout
  - get_order
generated: '2026-09-12'
method: generated
source: >-
  mcp/agilitassports-ucp-mcp-tools.json (live tools/list, 2026-09-12),
  https://agilitas.com/llms.txt, https://agilitas.com/robots.txt,
  https://agilitas.com/policies/refund-policy
---

# Complete an Agilitas purchase safely

This skill spends the buyer's money. Read all of it before the first call.

## The merchant's rule, in the merchant's words

From `https://agilitas.com/llms.txt` and repeated in `https://agilitas.com/robots.txt`:

> **Checkout requires human approval.** Agents must not complete payment without explicit buyer
> consent. If you cannot get contemporaneous buyer approval at the moment of payment, install
> https://shop.app/SKILL.md and route the purchase through Shop Pay instead.

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement automatically — no
> scripted form fills, browser automation, or end-to-end agent flows that finalize payment without an
> explicit, contemporaneous human approval step.

"Contemporaneous" is the operative word. Approval collected earlier in the session, or a standing
instruction to "just buy it", is not approval at the moment of payment.

## 1. Attach the payment instrument

`update_checkout` (or `create_checkout`) carries `checkout.payment.instruments[]`:

```json
{ "id": "...", "handler_id": "shopify.card", "type": "card",
  "billing_address": { "...": "..." },
  "credential": { "type": "...", "token": "..." },
  "selected": true }
```

The handlers this merchant declares in `https://agilitas.com/.well-known/ucp` are:

- `com.google.pay` (handler id `gpay`) — gateway `shopify`, cards VISA, MASTERCARD, AMEX, DISCOVER,
  billing address required
- `dev.shopify.card` (handler id `shopify.card`) — visa, master, american_express, discover,
  diners_club

The credential is the buyer's, passed through per call. You never hold a merchant credential.

## 2. Show the final numbers

Re-read the checkout and quote the total in major units before asking for approval. Minor units to
major: divide by 100 for INR/USD/EUR. Say the currency out loud.

## 3. Get approval, then complete — once

`complete_checkout` **requires** `meta["idempotency-key"]` alongside `meta["ucp-agent"].profile`.
Generate the key once, before the first attempt, and reuse the *same* key on every retry of that
same purchase. A new key on a retry is a second order.

```json
{
  "meta": {
    "ucp-agent": { "profile": "https://example.com/agent" },
    "idempotency-key": "<stable uuid for this purchase>"
  },
  "id": "gid://shopify/Checkout/..."
}
```

This is the only tool of the thirteen with an idempotency key, and it is the only one that needs one.

## 4. Confirm

`get_order` on the returned `gid://shopify/Order/{id}`. Give the buyer the order reference.

## 5. Tell the buyer the truth about undoing it

There is **no** refund, void or cancel tool in this API. Post-payment reversal is a human flow, and
Agilitas publishes real windows for it at `https://agilitas.com/policies/refund-policy`:

| Want to | Window | How |
|---|---|---|
| Cancel the order | any time **before dispatch** | return-and-exchange portal, account order history, or WhatsApp +91-8150863126 |
| Return | **within 14 days** of receiving the order, free of charge | same channels |
| Exchange | **within 14 days** of delivery, for another Lotto product on the site | same channels |

Conditions the buyer must be told: items unworn, unwashed, original packaging and tags intact —
broken or missing tag loops void eligibility. **Socks are final sale**, never returnable or
exchangeable. Cancellation applies to the whole order, not a single item. Products bought in offline
stores or other channels cannot be returned through this site.

Say that a cancellation or return is possible through those channels. Do **not** promise to do it
yourself — you have no tool that can.

## If something goes wrong

Quote the `x-request-id` response header. Be aware there is no developer support channel to quote it
to: `https://agilitas.com/pages/contact` is retail customer care.

Back off on `429`. The endpoint is rate-limited per IP and publishes no quota, so you cannot pace
from the response — you can only react.
