# Shipping Documents

- [Back to language README](README.md)

## Overview
- Retrieves documents generated after shipment processing.
- May return PDF, ZPL, attachments, and, when allowed, embedded documents.
- This module is published in v1.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Language header: `lang`.

## Conventions
- `attachments`, `embedded`, and `base64` accept booleans through the query string.
- `documents` receives a comma-separated list.

## Usage Flow
1. Create the shipment.
2. Wait for the status to indicate processing/completion.
3. Retrieve the documents by `shipping` or by channel.
4. Download the content in `url` or `base64`.

## Endpoints
### Retrieve documents by shipment
- Method and route: `GET /v1/shipping/{shipping}/docs`
- `shipping` accepts quotation UUID, AWB, or NF-e.

### Retrieve documents by channel
- Method and route: `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | yes | Shipment identifier | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Also accepts AWB and NF-e |
| `channel_identifier_number` | path | string | yes | Channel identifier | `334455` | - |
| `channel_code` | path | integer | no | Channel code | `1` | Optional |
| `attachments` | query | boolean | no | Includes attachments | `true` | Default `true` |
| `embedded` | query | boolean | no | Includes embedded docs | `true` | Default `true` |
| `base64` | query | boolean | no | Includes base64 | `true` | Default `true` |
| `documents` | query | string | no | Filters documents | `invoice_pdf,awb_pdf` | When empty, returns everything |

### Request example
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Response example
```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "carrier": "DHL - Teste",
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
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "string" }
    ]
  }
}
```

## Business Rules
- If there are no documents or attachments, the API may return `warning`.
- Embedded documents depend on a specific permission.
- The filterable document types are those listed in the shipment response.

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `400` | No document found | Confirm whether the shipment has already been processed |
| `500` | Unexpected error | Retry and contact the responsible team |

## Final Notes
- For integrations, retrieval by `shipping` is the most direct approach.
- Retrieval by channel is useful when the integrator stores the commercial identifier in the shipment.
