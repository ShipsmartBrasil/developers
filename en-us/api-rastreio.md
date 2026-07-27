# Shipment Tracking

## Overview
- Retrieves operational status and shipment history.
- Covers receiving, payment, processing, transport, and cancellation.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Language header: `lang`.

## Conventions
- `shipping` accepts UUID, AWB, or NF-e depending on the endpoint.
- Status codes use internal values such as `RCV0001`, `PPY0001`, `SPD0001`, `CCG0001`, and `CCD0001`.

## Response Formats
- The same endpoint returns `timeline` and `status`.
- `timeline` lists events in the order received from the carrier.
- `status` groups events by status using the platform's internal mapping.

## Usage Flow
1. Create the shipment.
2. Query status by `shipping` or by channel.
3. Track `received`, `processing_payment`, `processing_shipment`, `shipped`, and cancellation when applicable.

## Endpoints
### Status by shipment
- `GET /v1/shipping/{shipping}/status`

### Status by channel
- `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`
