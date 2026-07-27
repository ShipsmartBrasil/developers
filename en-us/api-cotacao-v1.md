# Quote

## Overview
- Generates international quotes with freight, insurance, and tax breakdowns.
- Use it before creating a shipment.
- The quote is the basis for shipment, tracking, and documents.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Language header accepted: `lang: pt|en|es`.

## Conventions
- `Content-Type: application/json`
- Dates use `Y-m-d H:i:s` in responses.
- Currency codes use ISO 4217.
- `level` controls the response detail: `simple` or `detailed`.

## Usage Flow
1. Send a quote with origin, destination, object type, and boxes/items.
2. Receive the `quotation` code returned by the API.
3. Fetch the result with `GET /v1/quotation/{quotation}/{level}`.
4. If eligible, reuse the same `quotation` to create the shipment.

## Functional Notes
- `simple` calculates by box with direct weight/value.
- `advanced` calculates with boxes and items distributed by `box_id`.
- `items` is accepted by validation, but the public OpenAPI contract does not expose it clearly.

## Eligibility for Shipment
- The quote must exist, not be expired, belong to the same API version, and not have been used yet.
- `package`/`pallet` with `simple` type are not eligible for shipment.
- Shipment also requires a compatible contract and allowed payment type.

## Endpoints
### Create quote
- Method and route: `POST /v1/quotation`
- Important rules: sender and receiver are required; `currency_quote` and `currency_payment` are required.

### Notes
- `object`: `doc`, `package`, `pallet`, `not_doc`
- `type`: `simple`, `advanced`, `items`
- `tax`: `sender` or `receiver`
- `measurement`: `metric` or `imperial`
