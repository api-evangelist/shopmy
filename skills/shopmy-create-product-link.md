---
name: Create a ShopMy product link
description: Resolve or search a product in the ShopMy catalog, check the creator's merchant rate, then create a trackable ShopMy link on the user's behalf.
api: openapi/shopmy-partners-openapi.yml
operations: [searchCatalog, fetchUrlRate, createLink]
scopes: [write_links]
---

# Create a ShopMy product link

Use this to turn a product URL or search term into a trackable ShopMy link for an authenticated creator.

## Auth
Every call needs BOTH headers:
- `Authorization: Bearer <developer key>`
- `X-ACCESS-TOKEN: Bearer <user access token>` (from the OAuth exchange; requires the `write_links` scope)

Base URL: `https://api.shopmy.us/v1/Partners`. HTTPS only.

## Steps
1. **Resolve the product** — `searchCatalog` (`GET /Catalog/search`). Pass either `search=<query>` OR `url=<product url>` (never both). Use `limit`/`page` to paginate (limit max 50).
2. **Check the rate (optional)** — `fetchUrlRate` (`POST /Urls`) with `{ "url": "<product url>" }` to get the creator's merchant rate before creating the link. A `400` means missing URL or no merchant for that URL.
3. **Create the link** — `createLink` (`POST /Links`) with `{ "url": "<product url>", "title": "<optional>" }`. Returns the short link (e.g. `https://go.shopmy.us/p-12345`).

## Errors
- `400` — bad/missing parameters (see errors/shopmy-problem-types.yml).
- `401` — missing scope: "Missing required scopes: The user has not authorized you to perform this action."
- `403` — action forbidden for this app/user.

Idempotency is not supported — do not blindly retry `createLink`; check `fetchLinks` first if a retry is needed.
