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

### Query shipment status
- `GET /v1/shipping/{shipping}/status`
- `shipping` accepts quote UUID, AWB, or 44-digit NF-e key.

### Query status by channel
- `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Query documents
- `GET /v1/shipping/{shipping}/docs`
- Returns documents, attachments, and embedded docs when allowed.

### Query documents by channel
- `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Update shipment
- `PUT /v1/shipping/{shipping}`
- Use it to attach documents and/or bind NF-e in exceptional cases.
- Attached files must be base64 and only `PDF`, `XML`, `JPEG`, or `PNG`.
- Maximum size is `5 MB` per file.

### Cancel shipment
- `DELETE /v1/shipping/{shipping}`
- `reason` is optional in the body and accepts free text.

## Example request
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Customer requested cancellation"}'
```

## Example response
```json
{
  "status": "success",
  "message": "Request processed, check the status shortly to follow the cancellation process"
}
```

## Common Errors
| Status | Meaning | Fix |
|---|---|---|
| `400` | Shipment not found or invalid flow | Check the identifier and the quote |
| `422` | Validation | Fix required fields |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- The status endpoint is the canonical source for the operational life cycle.
- The document endpoints are the fastest way to retrieve generated files.
