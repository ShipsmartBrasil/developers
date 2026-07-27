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

### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | yes | Shipment identifier | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Accepts quote UUID, AWB, and NF-e |
| `channel_identifier_number` | path | string | yes | Channel identifier | `334455` | - |
| `channel_code` | path | integer | no | Channel code | `1` | Optional |
| `attachments` | query | boolean | no | Include attachments | `true` | Default `true` |
| `embedded` | query | boolean | no | Include embedded docs | `true` | Default `true` |
| `base64` | query | boolean | no | Include base64 | `true` | Default `true` |
| `documents` | query | string | no | Filter documents | `invoice_pdf,awb_pdf` | Empty returns everything |

### Example request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: en'
```

### Example response
```json
{
  "status": "success",
  "message": "Shipment documents found!",
  "data": {
    "carrier": "DHL - Test",
    "tracking_id": "000000",
    "docs": {
      "Invoice": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "string"
      },
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    },
    "embedded_docs": [
      { "type": "pdf", "base64": "string" }
    ],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Invoice", "base64": "string" }
    ]
  }
}
```

## Business Rules
- If there are no documents or attachments, the API may return `warning`.
- Embedded documents depend on specific permission.
- Filterable document types are the ones listed in the shipment response.

## Common Errors
| Status | Meaning | Fix |
|---|---|---|
| `400` | No documents found | Confirm the shipment has already been processed |
| `500` | Unexpected error | Retry and contact the responsible team |

## Final Notes
- Querying by `shipping` is the most direct approach for integrations.
- Querying by channel is useful when the integrator stores the commercial identifier in the shipment.
