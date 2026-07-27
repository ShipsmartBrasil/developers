# Box

## Overview
- Manages box registration and lookup for quotes, drafts, and shipments.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Optional language header: `lang`.

## Conventions
- `Content-Type: application/json`
- Lookup uses `boxes[box_ids][]` in the query string.
- Create/update is asynchronous.

## Usage Flow
1. List available boxes.
2. Register or update base boxes.
3. Remove boxes when needed.
4. Reuse the codes in quotes and drafts.

## Endpoints
### List boxes
- `GET /v1/box`

### Create or update boxes
- `POST /v1/box`

### Delete boxes
- `DELETE /v1/box`
