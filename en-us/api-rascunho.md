# Draft

## Overview
- Creates, lists, updates, and removes shipment drafts.
- Drafts are processed asynchronously and can evolve into quote + shipment.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`

## Authentication
- Bearer token in `Authorization`.
- Language header: `lang`.

## Conventions
- Drafts accept partial payloads.
- Draft status can be `received`, `edited`, `validating`, `incomplete`, `complete`, `processing`, or `error`.

## Usage Flow
1. Send the draft with the data you already have.
2. Wait for asynchronous processing.
3. List or query drafts.
4. Update only `incomplete` drafts.
5. Remove drafts when needed.

## Endpoints
### Create draft
- `POST /v1/draft`

### Update draft
- `PUT /v1/draft/{code}`

### List drafts
- `GET /v1/draft`

### Delete draft
- `DELETE /v1/draft`
