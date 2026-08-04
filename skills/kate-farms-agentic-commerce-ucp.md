---
name: Transact with Kate Farms over UCP / MCP
description: Discover the Kate Farms Universal Commerce Protocol merchant profile, understand the agent-profile gate on its MCP endpoint, and run the five-tool buying flow the merchant itself publishes.
api: mcp/kate-farms-mcp.yml
endpoint: https://shop.katefarms.com/api/ucp/mcp
operations: [search_catalog, create_cart, create_checkout, update_checkout, complete_checkout]
generated: '2026-08-04'
method: generated
---

# Transact with Kate Farms over UCP / MCP

Kate Farms publishes a Universal Commerce Protocol merchant profile and an MCP endpoint for
agent-driven purchase. This is the surface the merchant *intends* agents to use — the GraphQL
path is the fallback when you cannot enrol.

## 1. Discover

```
GET https://shop.katefarms.com/.well-known/ucp
```

Returns the merchant profile (captured verbatim at `well-known/kate-farms-ucp.json`):

- UCP versions `2026-04-08` (current) and `2026-01-23`
- MCP transport endpoint `https://shop.katefarms.com/api/ucp/mcp`
- Capabilities: catalog search + lookup, cart, discount, fulfillment, checkout, order
- Payment handlers: Google Pay (`gpay`), Shopify Card (`shopify.card`), Shop Pay (`shop_pay`)
- Fulfillment config: single shipping destination only, no method combinations

## 2. Know the gate before you call

`tools/list` is **not anonymous**. Posting

```json
{"jsonrpc":"2.0","id":1,"method":"tools/list"}
```

returns HTTP 422 with

```json
{"code":-32001,"message":"UCP discovery failed",
 "data":{"code":"invalid_profile_url","content":"Unable to fetch agent profile: Missing profile uri"}}
```

You must present a resolvable **UCP agent profile URI**. Until you do, no tool input schemas are
available. Bearer tokens for the protected resource `https://shop.katefarms.com` come from the
authorization server declared at `/.well-known/oauth-protected-resource`
(`https://shopify.com/authentication/2056802`), presented in the `Authorization` header.

## 3. The published flow

Kate Farms documents this sequence in its own `agents.md`:

1. `search_catalog` — find products matching buyer intent
2. `create_cart` — add the desired items
3. `create_checkout` — start the purchase
4. `update_checkout` — set shipping address and method
5. `complete_checkout` — finalize, **buyer must approve**

Pass `context.address_country` and `context.currency` on every call for correct pricing and
availability.

## 4. Rules the merchant states

- **Checkout requires contemporaneous human approval.** If you cannot get buyer approval at the
  moment of payment, Kate Farms directs you to install `https://shop.app/SKILL.md` and route the
  purchase through Shop Pay instead.
- **The MCP endpoint is rate-limited per IP.** Back off on 429.
- **Do not screen-scrape the storefront.** Use this endpoint, the Shop skill, or the read-only
  JSON endpoints listed in `llms/kate-farms-agents.md`.

## 5. If you cannot enrol

Fall back to the anonymous Storefront GraphQL API — `kate-farms-browse-catalog.md` and
`kate-farms-build-cart-and-checkout.md`. `mcp/kate-farms-tool-crosswalk.yml` maps each UCP tool
onto the GraphQL fields that back the same capability, and records what the two surfaces do
*not* share: content (articles, pages, metaobjects), shopper account management, and Shop Pay
session handling are GraphQL-only; the buyer-approval invariant and payment-handler negotiation
are MCP-only.
