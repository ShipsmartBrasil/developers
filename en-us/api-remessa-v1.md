# Shipment

- [Back to English README](README.md)

## Overview
- Creates, updates, cancels, and queries shipments.
- Also exposes document lookup and status/tracking.
- This is the most complete legacy shipping flow.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Optional language header: `lang`.

## Conventions
- `Content-Type: application/json`
- Shipment uses `quotation` as the link to the quote.
- The `quotation` must belong to the same shipment API version.
- `payment.type` defines the financial model.

## Usage Flow
1. Create the quote.
2. Check whether it is eligible for shipment.
3. Send the shipment with the returned `quotation`.
4. Check `status` until processing is complete.
5. Download documents from `/docs`.
6. Schedule pickup if needed.

## Business Rules
- `package`/`pallet` quotes with `simple` type are not eligible.
- The quote cannot be expired.
- The shipment cannot already exist for the same quote.
- Payment type must be enabled for the user.
- Once created, the shipment is irreversible in normal operation.
- Later changes are only allowed in exceptional cases: invoice number when it was not sent, and attachment upload when files were not included at creation.

## Endpoint
### Create shipment
- Method and route: `POST /v1/shipping`

### Notes
- `quotation` must be from v1.
- `carrier_code` must exist for the quote.
- `attachments` accepts PDF, XML, JPEG, and PNG.
