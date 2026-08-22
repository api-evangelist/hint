---
name: hint-cart-to-checkout
description: >-
  Build a cart and drive a checkout to completion on the Hint (drinkhint.com) storefront over the
  store's Model Context Protocol endpoint, with the buyer approval and idempotency rules the
  provider requires — and the knowledge that payment cannot be reversed afterwards.
generated: '2026-08-22'
method: generated
source: mcp/hint-mcp-tools.json (live tools/list, 2026-08-22) + https://www.drinkhint.com/agents.md
api: Hint Agentic Commerce (UCP / MCP)
endpoint: https://www.drinkhint.com/api/ucp/mcp
transport: json-rpc-2.0 over HTTPS
auth: none
operations:
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

# Buying from Hint on a buyer's behalf

This skill writes real state and, at the last step, takes real money. Read the two hard rules
before the steps.

## Hard rule 1 — payment needs a human

Hint's published agent instructions and `robots.txt` both say it: **do not complete checkout,
payment, or order placement automatically.** No scripted form fills, no end-to-end flow that
finalizes payment without an explicit, contemporaneous human approval step. If you cannot get that
approval at the moment of payment, do not call `complete_checkout` — route the purchase through
the Shop skill at `https://shop.app/SKILL.md` instead, which carries buyer approval itself.

## Hard rule 2 — completion is irreversible

`cancel_cart` and `cancel_checkout` exist and work *before* payment. After `complete_checkout`
succeeds there is no reversal tool on this surface, and Hint's published refund policy states the
company does not do returns, exchanges or refunds, directing buyers to customer service. Treat
`complete_checkout` as a one-way door and say so to the buyer before they approve.

## Steps

1. **Build the cart** — `create_cart` with `cart.line_items[]` (`{id, quantity, item}`).
   Set `cart.buyer.email` / `.phone_number` and `cart.context` (`address_country`, `currency`,
   `postal_code`, `language`) so pricing and availability are correct.
2. **Adjust** — `update_cart` with the cart `id` to change quantities, fulfillment or discounts.
   `cart.discounts.codes[]` is case-insensitive and **replaces** previously applied codes rather
   than adding to them. Only prompt for a discount code if the buyer raises one.
3. **Read back** — `get_cart` to confirm contents and totals before moving on.
4. **Convert to a checkout** — `create_checkout`, passing `checkout.cart_id` to carry the cart
   across. Totals, taxes and applicable discounts come back on the checkout.
5. **Fulfillment and buyer details** — `update_checkout` to set `checkout.fulfillment.methods[]`
   and complete `checkout.buyer`. This store allows a **single shipping destination** and shipping
   only; multi-destination is declared unsupported in its UCP profile.
6. **Show the buyer the total and get approval.** Convert minor units to major units first
   (`{"amount": 2500, "currency": "USD"}` is $25.00). State that the purchase cannot be refunded.
7. **Complete** — `complete_checkout` with the checkout `id`. This call **requires**
   `meta["idempotency-key"]`; generate one stable key per completion attempt and reuse the same key
   on any retry, so a network failure cannot double-charge. It is the only tool on the surface that
   requires an idempotency key.
8. **Confirm** — `get_order` with the returned order identifier, and give the buyer the confirmation.

## Backing out

- Buyer changes their mind before payment: `cancel_checkout` (or `cancel_cart` if no checkout
  exists yet). No time window is published for either.
- Buyer changes their mind after payment: there is nothing you can call. Direct them to
  `https://www.drinkhint.com/pages/contact-us`.

## Conventions and errors

`meta["ucp-agent"].profile` is required on every call. Errors are JSON-RPC error objects with a
branchable `error.data.code` and a `continue_url` for handing the buyer back to the storefront.
Back off on `429`. Full detail in `conventions/hint-conventions.yml`,
`errors/hint-problem-types.yml` and `rate-limits/hint-rate-limits.yml`.
