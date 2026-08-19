---
name: Report a ShopMy order for commission attribution
description: >-
  Send a completed order to ShopMy so a creator commission is attributed, then
  keep it accurate through returns, edits and cancellations. Covers the three
  server-to-server tracking routes, the outcome codes that decide whether
  attribution actually happened, and the safe-retry rules.
api: asyncapi/shopmy-tracking-events.yml
grounding: docs
grounding_note: >-
  ShopMy publishes no OpenAPI for the tracking routes, so this skill is grounded
  in the documented request contracts on docs.shopmy.us rather than in
  operationIds. Every field, status and outcome code below is quoted from those
  pages — none is inferred.
operations: []
routes:
  - POST https://api.shopmy.us/api/order_confirmation
  - POST https://api.shopmy.us/api/Affiliates/update
  - POST https://api.shopmy.us/api/Affiliates/cancel
docs:
  - https://docs.shopmy.us/reference/order-confirmation
  - https://docs.shopmy.us/reference/order-update
  - https://docs.shopmy.us/reference/order-cancellation
  - https://docs.shopmy.us/reference/tracking-routes-overview
---

# Report a ShopMy order for commission attribution

Use this when a brand runs a direct (non-Shopify) integration and must report its
own orders to ShopMy. If the brand is on Shopify, **do not call these routes** —
the Shopify app already reports orders, and a confirmation you send will come
back with the `shopify_handoff` outcome.

## Before you start

- The three routes sit on `https://api.shopmy.us/api`, **not** on the Partners API
  base `https://api.shopmy.us/v1/Partners`. Different base path, different auth.
- Read `errors/shopmy-outcome-codes.yml` first. It is the contract that matters
  here, more than the HTTP status.

## Step 1 — Confirm the order

`POST https://api.shopmy.us/api/order_confirmation`

Call this from the order confirmation page or server **every time an order
completes**, not only when you believe a creator was involved. ShopMy decides
attribution; your job is to report every order.

No API key. ShopMy matches the event to the brand through the domain in
`page_url`, so that field is load-bearing — get it wrong and the event is
orphaned.

Required: `orderAmount` (a plain number greater than 0 — no currency symbols, no
HTML), `orderId` (your unique id for the order), `currency` (ISO code),
`page_url`.

Strongly recommended: `clickId`, read from the shopper's `sms_click_id` cookie.
The docs call it "the strongest attribution signal" — without it ShopMy falls
back to code and shopper matching, and attribution rates drop.

Optional: `code` (discount code applied), `is_returning_customer`, `domain`
(explicit override when `page_url` cannot carry it), `returnWindowClosesAt`,
and `test` — set `test: true` while integrating so events are counted separately
and never billed.

```json
{
  "orderAmount": 129.5,
  "orderId": "1001234",
  "currency": "USD",
  "page_url": "https://yourstore.com/checkout/thanks",
  "clickId": "<sms_click_id cookie value>",
  "code": "CREATOR15"
}
```

## Step 2 — Read the outcome, not the status code

This is where integrations go wrong. A `200` means **the event was recorded**,
including orders ShopMy had nothing to attribute. It does not mean a commission
was created.

- `tracked` — a commission was created. This is the only success.
- `no_attribution` — a direct order with no ShopMy involvement. Expected for most
  of your traffic. Do nothing.
- `shopify_handoff`, `blacklisted_code`, `expired_click`, `cross_brand_click`,
  `property_mismatch`, `banned_user` — informational. No action.
- `insert_failed`, `handler_error` — ShopMy-side failures. Do not change your
  integration; do not treat these as your bug.

Treat these as defects in **your** integration and fix them:

- `invalid_order_amount` — send the total as a plain number.
- `invalid_order_id` — send a unique order id.
- `malformed_click_id` — forward the `sms_click_id` cookie value exactly as stored.
- `unresolved_domain` — `page_url` (or `domain`) must match the domain registered
  with ShopMy.
- `unregistered_code` — the discount code is not attached to any creator. Add it
  to the creator's account if it should track.
- `click_not_found` — you sent a click id with no matching click on file.

Every outcome is also visible per brand on the Tracking Health page at
`https://shopmy.us/tracking-health`. Use it to verify a new integration before
trusting it.

## Step 3 — Keep the amount accurate

`POST https://api.shopmy.us/api/Affiliates/update`

Call on any partial return, edited line item or adjusted discount. Unlike Step 1,
this route **is authenticated**:

```
Authorization: Bearer <your developer key>
```

Required: `order_id` (the same id you sent on the original confirmation),
`new_order_amount`, `currency`. Optional: `domain`, `test`.

`new_order_amount` is an **absolute new total**, not a delta.

```json
{ "order_id": "1001234", "new_order_amount": 89.0, "currency": "USD" }
```

`order_not_found` comes back as a success — it usually just means the original
order carried no ShopMy attribution, so there is no commission to adjust.

## Step 4 — Cancel when an order dies

`POST https://api.shopmy.us/api/Affiliates/cancel`

Same bearer auth. Required: `order_id`. Optional: `domain`, `test`.

```json
{ "order_id": "1001234" }
```

Call this on a cancellation or a full refund so the creator is never paid on a
sale that did not stick. Partial refunds go through Step 3 instead.

## Retry rules

The tracking routes are safe to retry, keyed on your order id:

- Re-sending a confirmation for an id ShopMy already accepted returns
  `duplicate_order` with HTTP `400`. The first event counted; the retry is
  ignored. You will not double-count a commission.
- Re-cancelling returns `already_cancelled` with HTTP `200` and no further effect.
- Update sets an absolute amount, so replaying the same update converges.

There is no `Idempotency-Key` header — your `orderId` is the key. Use a stable,
unique id per order and retries are safe. See
`conventions/shopmy-conventions.yml`.

## What you cannot do

- **You cannot subscribe to ShopMy events.** There are no outbound webhooks. To
  learn about commissions, poll `fetchOrderReport` on the Partners API — capped
  at 200 requests/day (`rate-limits/shopmy-rate-limits.yml`).
- **There is no rate-limit header.** No route returns `RateLimit-*` or
  `Retry-After`, so back off on your own schedule.
- **Authentication is asymmetric.** Order Confirmation is unauthenticated and
  identified only by domain; Update and Cancellation require the developer key
  and return `401` without it. Do not assume one credential covers all three.
