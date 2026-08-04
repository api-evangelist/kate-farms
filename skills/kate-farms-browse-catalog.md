---
name: Browse the Kate Farms catalog
description: Find Kate Farms formulas and shakes, resolve a purchasable variant, and read its price, case size and quantity rules — all anonymously, with no credential.
api: graphql/kate-farms-storefront.graphql
endpoint: https://shop.katefarms.com/api/2026-01/graphql.json
operations: [products, search, predictiveSearch, collection, collections, productByHandle, productRecommendations]
generated: '2026-08-04'
method: generated
---

# Browse the Kate Farms catalog

Kate Farms sells plant-based medical nutrition formulas and shakes. The public catalog is
readable through the Storefront GraphQL API on their own host with **no authentication**.

## Endpoint

```
POST https://shop.katefarms.com/api/2026-01/graphql.json
Content-Type: application/json
```

The current stable version is `2026-07`; four calendar versions are supported at a time
(see `lifecycle/kate-farms-lifecycle.yml`). Read `x-shopify-api-version` on the response to
confirm which one served you, and `x-request-id` for support.

## Steps

1. **Search** with the `search` or `predictiveSearch` root field for free-text buyer intent
   ("peptide 1.5 vanilla", "pediatric standard"). Use `products` with `query:` for structured
   filtering, or `collection(handle:)` to walk a merchandised group such as tube-feeding formulas.
2. **Resolve a variant.** Nothing is purchasable at the `Product` level. Read `variants` (or
   `selectedOrFirstAvailableVariant`) to get a `ProductVariant`, which is the SKU — flavour plus
   case size.
3. **Read the buying constraints before you build a cart.** On the variant:
   - `availableForSale`
   - `quantityRule { minimum maximum increment }` — Kate Farms sells by the case, so this is
     load-bearing and is the most common source of cart errors.
   - `price { amount currencyCode }` and `unitPrice` / `unitPriceMeasurement`
   - `sellingPlanAllocations` — if the variant is subscription-only you will need a selling plan
     (`VARIANT_REQUIRES_SELLING_PLAN`).
4. **Paginate with cursors.** Every list is a Relay connection: pass `first`/`after`, read
   `pageInfo { hasNextPage endCursor }`. There is no page-number pagination.

## Example

```graphql
query FindFormula($q: String!) {
  products(first: 10, query: $q) {
    pageInfo { hasNextPage endCursor }
    edges { node {
      id title handle availableForSale
      variants(first: 10) { edges { node {
        id title sku availableForSale
        price { amount currencyCode }
        quantityRule { minimum maximum increment }
        sellingPlanAllocations(first: 5) { edges { node { sellingPlan { id name } } } }
      } } }
    } }
  }
}
```

## Rules

- Costs are metered. Every response carries `extensions.cost.requestedQueryCost` — keep selections
  narrow. Back off on `429`.
- Errors come back as HTTP 200 with an `errors[]` array; business failures are typed
  `userErrors[]` with enum codes. See `errors/kate-farms-problem-types.yml`.
- A lighter read-only path exists if you do not need GraphQL:
  `GET /products/{handle}.json` and `GET /collections/{handle}/products.json`, documented by
  Kate Farms in `llms/kate-farms-agents.md`.
