# Shipping Tracking

- [Back to language README](README.md)

## Overview
- Retrieves the operational status and history of a shipment.
- Shows receipt, payment, processing, transport, and cancellation.
- Associated with the main operational flow.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Language header: `lang`.

## Conventions
- The `shipping` field accepts UUID, AWB, or NF-e, depending on the endpoint.
- Statuses use internal codes such as `RCV0001`, `PPY0001`, `SPD0001`, `CCG0001`, and `CCD0001`.

## Response Formats
- The same endpoint returns two formats: `timeline` and `status`.
- `timeline` brings the events in the order they arrive from the carrier.
- `status` brings events grouped by status, using the platform's internal mapping to simplify reading.

## Usage Flow
1. Create the shipment.
2. Check the status by `shipping` or by channel.
3. Track `received`, `processing_payment`, `processing_shipment`, `shipped`, and, if applicable, cancellation.

## Endpoints
### Status by shipment
- Method and route: `GET /v1/shipping/{shipping}/status`

### Status by channel
- Method and route: `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | yes | Shipment identifier | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Accepts UUID, AWB, and NF-e |
| `channel_identifier_number` | path | string | yes | Channel number | `334455` | - |
| `channel_code` | path | integer | no | Channel code | `1` | Optional |

### Request example
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Response example
```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Fedex Economy Startup Test BR",
      "tracking_number": "271249639538",
      "timeline": [
        {
          "update_date": "2024-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
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
- Payment must exist for the shipment to return full status.
- If payment has not yet been approved, the status may remain in `processing_payment`.
- When the shipment is billed and has not yet been processed by the carrier, `processing_shipment` may appear.
- When there is cancellation, the response may include `cancelling` or `cancelled`.

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `400` | Shipment not found | Check identifier |
| `404` | No linked payment | Confirm shipment processing |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- Tracking is the most complete way to monitor the shipment lifecycle.
- The timeline is derived from internal events and the payment/carrier response.
- Status grouping is an operational summary to make the shipment tracking easier to understand.
