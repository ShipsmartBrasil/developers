# Shipment Documents

- [Back to English README](README.md)

## Overview
- Retrieves documents generated after shipment processing.
- Can return PDF, ZPL, attachments, and, when allowed, embedded documents.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Language header: `lang`.

## Conventions
- `attachments`, `embedded`, and `base64` accept booleans in the query string.
- `documents` accepts a comma-separated list.

## Usage Flow
1. Create the shipment.
2. Wait until processing is complete.
3. Query documents by `shipping` or by channel.
4. Download content via `url` or `base64`.

## Endpoints
### Query by shipment
- `GET /v1/shipping/{shipping}/docs`

### Query by channel
- `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`
