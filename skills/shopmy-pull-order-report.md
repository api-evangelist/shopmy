---
name: Pull a ShopMy brand order report
description: As a Brand Partner, page through affiliate order records for your brand across all network sources, filtered by transaction or record-update date.
api: openapi/shopmy-partners-openapi.yml
operations: [fetchOrderReport]
scopes: []
---

# Pull a ShopMy brand order report

Use this as a Brand Partner to reconcile affiliate orders attributed to your brand across all ShopMy network sources.

## Auth
Single header (no OAuth user token needed):
- `Authorization: Bearer <developer key>` (Brand Account Settings > Tokens > Developer Key)

Base URL: `https://api.shopmy.us/v1/Partners`. HTTPS only.

## Steps
1. **Request the report** — `fetchOrderReport` (`POST /OrderReport`) with `{ "domain": "<your brand domain>" }` (required).
2. **Filter** — add `transactionStartDate`/`transactionEndDate` or `recordUpdatedStartDate`/`recordUpdatedEndDate` (format `YYYY-MM-DD HH:mm:ss` UTC); `source` to filter by network (e.g. `shopmyshelf`); `reportBySKU=true` to break out SKUs.
3. **Paginate** — `limit` (default/max 500) and `page` (zero-indexed). Orders come back sorted by transaction date descending.

## Rate limits & errors
- **200 requests/day** (including paginated requests) — at 500 orders/call that is up to 100,000 orders/day. Exceeding returns `429`.
- `401` — unauthorized, e.g. requesting a `source` you cannot access.
- Use the sandbox (`Fetch Order Report Sandbox`, 1000 req/day) to test — see sandbox/shopmy-sandbox.yml.

Business windows: 30-day cookie window; 30-day return window before amounts lock (see lifecycle/shopmy-lifecycle.yml).
