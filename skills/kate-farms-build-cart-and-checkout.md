---
name: Build a Kate Farms cart and take it to checkout
description: Create a cart, add case-quantity-valid lines, set buyer identity and a US delivery address, select a delivery option, and hand the buyer a checkout — with the retry rules that matter because cart mutations are not idempotent.
api: graphql/kate-farms-storefront.graphql
endpoint: https://shop.katefarms.com/api/2026-01/graphql.json
operations: [cartCreate, cartLinesAdd, cartLinesUpdate, cartLinesRemove, cartBuyerIdentityUpdate, cartDeliveryAddressesAdd, cartSelectedDeliveryOptionsUpdate, cartDiscountCodesUpdate, cartPrepareForCompletion, cartSubmitForCompletion, cartCompletionAttempt, cart]
generated: '2026-08-04'
method: generated
---

# Build a Kate Farms cart and take it to checkout

## Preconditions

Resolve a real `ProductVariant` id first (see `kate-farms-browse-catalog.md`) and respect its
`quantityRule`. Kate Farms ships within the United States; a non-US or unserviceable postal code
fails with `ZIP_CODE_NOT_SUPPORTED`.

## Steps

1. **`cartCreate`** — create the cart, optionally with `lines` inline. Keep the returned
   `cart.id` and `cart.checkoutUrl`.
2. **`cartLinesAdd` / `cartLinesUpdate` / `cartLinesRemove`** — maintain lines. Attach
   `sellingPlanId` for subscription variants.
3. **`cartBuyerIdentityUpdate`** — set at minimum `email`, plus `countryCode` so pricing and
   availability resolve. A missing email fails submission with `BUYER_IDENTITY_EMAIL_REQUIRED`.
4. **`cartDeliveryAddressesAdd`** — add the shipping address. This store allows only a single
   delivery destination (`allows_multi_destination.shipping: false` in the UCP profile).
5. **Wait for delivery groups.** Re-query `cart { deliveryGroups }` until they resolve; calling
   on before they do yields `PENDING_DELIVERY_GROUPS`.
6. **`cartSelectedDeliveryOptionsUpdate`** — pick a delivery option. Skipping this fails
   submission with `NO_DELIVERY_GROUP_SELECTED`.
7. **`cartDiscountCodesUpdate`** — optional; apply a promotion code.
8. **Hand off or submit.**
   - Simplest and safest: give the buyer `cart.checkoutUrl` and let them finish in Shopify checkout.
   - Programmatic: `cartPrepareForCompletion`, then `cartSubmitForCompletion`, then poll
     `cartCompletionAttempt(attemptId:)` until it resolves.

## Idempotency and retries — read this before you retry anything

**Cart mutations take no idempotency key.** The only idempotency-keyed operation in the entire
schema is `shopPayPaymentRequestSessionSubmit`. So:

- Never blind-retry `cartSubmitForCompletion`. Re-read the cart, or poll
  `cartCompletionAttempt`, and only resubmit if the attempt is genuinely absent.
- `SERVICE_UNAVAILABLE` and `PAYMENT_TRANSIENT_ERROR` are the retryable failures. Back off first.
- `PAYMENT_CARD_DECLINED`, `PAYMENT_CALL_ISSUER` and `PAYMENT_INSUFFICIENT_FUNDS` are terminal —
  return them to the buyer and ask for another payment method. Do not retry.
- `INVENTORY_RESERVATION_ERROR` is an availability problem, not a payment problem; re-check
  availability before resubmitting.

## Error handling

Every mutation returns a `userErrors[]` array with a typed enum `code`. Check it even on HTTP 200.
The full catalogue with remediation is in `errors/kate-farms-problem-types.yml`. The ones you will
actually hit: `MINIMUM_NOT_MET`, `MAXIMUM_EXCEEDED`, `INVALID_INCREMENT` (case-size rules),
`VARIANT_REQUIRES_SELLING_PLAN`, `ZIP_CODE_NOT_SUPPORTED`, `PENDING_DELIVERY_GROUPS`.

## Buyer consent

If you are acting for a human, **do not complete payment without contemporaneous buyer approval.**
Kate Farms states this explicitly in its own agent document
(`llms/kate-farms-agents.md`): checkout requires human approval.
