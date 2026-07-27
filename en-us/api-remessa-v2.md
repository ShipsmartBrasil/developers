# Shipment V2

- [Back to English README](README.md)

## Overview
- Creates shipments with a simplified contract.
- The analyzed source does not expose status, documents, cancellation, or update endpoints for v2.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Language header: `lang`.

## Conventions
- `Content-Type: application/json`
- `quotation` must be created in v2.
- `payment.type` still defines the financial flow.

## Usage Flow
1. Create the quote in v2.
2. Send the shipment with the returned `quotation`.
3. Follow the rest of the flow through v1 endpoints when applicable.

## Endpoint
### Create shipment
- Method and route: `POST /v2/shipping`
- Important rule: the quote must be v2.

### Main differences from v1
- v2 keeps only the creation endpoint in the public documentation.
- `attachments` in v2 accepts only PDF or XML in the analyzed source.
- The contract is stricter for address fields.
