# Items

## Overview
- Manages items used in quotes, drafts, and shipments.
- Items can be queried, created, updated, and deleted in bulk.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Optional language header: `lang`.

## Conventions
- `Content-Type: application/json`
- Lookup uses `items[skus][]` in the query string.
- Create/update/delete are asynchronous.

## Usage Flow
1. List already registered items.
2. Register base items.
3. Update items when dimensions, weight, or classification change.
4. Remove obsolete items.

## Endpoints
### List items
- `GET /v1/item`

### Create items
- `POST /v1/item`

### Update items
- `PUT /v1/item`

### Delete items
- `DELETE /v1/item`
