---
name: hint-catalog-discovery
description: >-
  Search, filter and inspect products in the Hint (drinkhint.com) storefront catalog over the
  store's anonymous Model Context Protocol endpoint, and quote prices to a buyer correctly.
generated: '2026-08-22'
method: generated
source: mcp/hint-mcp-tools.json (live tools/list, 2026-08-22) + https://www.drinkhint.com/agents.md
api: Hint Agentic Commerce (UCP / MCP)
endpoint: https://www.drinkhint.com/api/ucp/mcp
transport: json-rpc-2.0 over HTTPS
auth: none
operations:
  - search_catalog
  - lookup_catalog
  - get_product
---

# Discovering Hint products

Read-only. Nothing in this skill spends money or creates state.

## Before you call anything

Every tool on this endpoint requires `meta["ucp-agent"].profile` — a resolvable https URI that
describes you, the calling agent. Omit it and the server answers HTTP 422 with JSON-RPC error
`-32001` / `invalid_profile_url`. It is not a credential; there is no API key and no signup.

Pass buyer context on every catalog call so pricing and availability are right:
`catalog.context.address_country` (ISO 3166-1 alpha-2) and `catalog.context.currency` (ISO 4217),
plus `address_region`, `postal_code` and `language` when you know them.

## Steps

1. **Search** — call `search_catalog` with `catalog.query` set to the buyer's words. Narrow with
   `catalog.filters.categories[]` (OR logic), `catalog.filters.price.min` / `.max` in **minor
   currency units**, and `catalog.filters.available` (defaults to `true`, sale-ready items only).
2. **Page** — results are cursor-paged. Send `catalog.pagination.limit` (default `10`, minimum `1`)
   and carry the returned cursor back in `catalog.pagination.cursor`. Do not try offset paging.
3. **Resolve a set of known items** — when you already hold identifiers, call `lookup_catalog` with
   `catalog.ids[]` instead of searching again. It is not paginated.
4. **Get full detail before recommending** — call `get_product` with `catalog.id`, and
   `catalog.selected[]` (`{name, label}` pairs) to pin a specific variant such as a flavor or pack
   size.

## Quoting a price

Prices come back as integers in the currency's ISO 4217 **minor units** paired with a currency
code: `{"amount": 600, "currency": "USD"}` is $6.00. Divide by 100 for two-decimal currencies
before you say a number to a buyer. Zero-decimal currencies such as JPY are already whole units.

## Errors

The envelope is the JSON-RPC error object with a machine-readable `error.data.code` to branch on
and an `error.data.continue_url` to hand the buyer back to the storefront. See
`errors/hint-problem-types.yml`.

## Rate limits

The endpoint is rate-limited per IP and publishes no number. Back off exponentially on `429`. The
`shopify-complexity-score` response header is the only per-request cost signal returned. See
`rate-limits/hint-rate-limits.yml`.
