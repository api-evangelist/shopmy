---
name: Build and edit a ShopMy collection
description: Create a public ShopMy shelf collection for a creator, then edit its name, description, or attached social links.
api: openapi/shopmy-partners-openapi.yml
operations: [fetchCollections, createCollection, editCollection]
scopes: [write_collections]
---

# Build and edit a ShopMy collection

Use this to create and maintain a creator's ShopMy shelf collections. Partner-created collections are always public; collection privacy cannot be changed via the API.

## Auth
Both headers required:
- `Authorization: Bearer <developer key>`
- `X-ACCESS-TOKEN: Bearer <user access token>` (requires `write_collections`)

Base URL: `https://api.shopmy.us/v1/Partners`.

## Steps
1. **List existing collections** — `fetchCollections` (`GET /Collections`, needs `read_collections` OR `write_collections`) to avoid duplicates.
2. **Create** — `createCollection` (`POST /Collections`) with `{ "name": "<required>", "description": "<optional>", "social_links": ["https://www.instagram.com/example"], "section_id": <optional> }`. Each social link must be an HTTPS Instagram, TikTok, or YouTube URL. Omitting `section_id` places it in the user's first standard shop section.
3. **Edit** — `editCollection` (`PUT /Collections/{id}`) with any of `name`, `description`, `social_links`, `section_id`.

## Errors
- `400` — missing name, invalid section, or an invalid (non Instagram/TikTok/YouTube) social link.
- `401` — missing `write_collections` scope.

See conventions/shopmy-conventions.yml and errors/shopmy-problem-types.yml.
