# Shipment Tracking

- [Back to English README](README.md)

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

### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | yes | Shipment identifier | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Accepts UUID, AWB, and NF-e |
| `channel_identifier_number` | path | string | yes | Channel number | `334455` | - |
| `channel_code` | path | integer | no | Channel code | `1` | Optional |

### Example request
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: en'
```

### Example response
```json
{
  "status": "success",
  "message": "Shipment status found!",
  "data": {
    "carrier": {
      "name": "Fedex Economy Startup Test BR",
      "tracking_number": "271249639538",
      "timeline": [
        {
          "update_date": "2024-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Shipment was received in our system"
        }
      ]
    },
    "received": [],
    "processing_payment": [],
    "shipped": []
  }
}
```

## Business Rules
- Payment must exist for the shipment to return the full status.
- If payment is not approved yet, status can remain `processing_payment`.
- When shipped but not yet processed by the carrier, it can show `processing_shipment`.
- Cancellation can return `cancelling` or `cancelled`.

## Common Errors
| Status | Meaning | Fix |
|---|---|---|
| `400` | Shipment not found | Check the identifier |
| `404` | No linked payment | Confirm shipment processing |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- Tracking is the most complete way to monitor the shipment lifecycle.
- `timeline` is derived from internal events and carrier/payment responses.
- Status grouping is an operational summary to make tracking easier.
