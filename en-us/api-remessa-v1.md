# Shipping

- [Back to language README](README.md)

## Overview
- Creates, updates, cancels, and retrieves shipments.
- Also exposes document and status/tracking retrieval.
- It is the most complete legacy version of the shipping flow.
## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Optional language header: `lang`.

## Conventions
- `Content-Type: application/json`
- The shipment uses `quotation` as the link to the quotation.
- The `quotation` must be from the same shipping API version.
- `payment.type` defines the financial model of the shipment.

## Usage Flow
1. Generate the quotation.
2. Check whether it is eligible for shipping.
3. Send the shipment with the returned `quotation`.
4. Check `status` until the shipment is processed.
5. Download the documents in `/docs`.
6. If necessary, schedule pickup.

## Business Rules
- `package`/`pallet` quotation with `simple` type is not eligible for shipping.
- The quotation cannot be expired.
- The shipment cannot have been previously created for the same quotation.
- The payment type must be enabled for the user.
- For DHL with a PJ sender in Brazil, `nfkey` may be required.
- After creation, the shipment is irreversible in normal operation.
- Later changes are only allowed in exceptional cases: sending the invoice number when it was not provided and sending documents to attach to the shipment when they were not sent during creation.

## Endpoints
### Create shipment
- Method and route: `POST /v1/shipping`
- When to use: after obtaining an eligible quotation.

#### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | yes | Quotation ID | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Must be from the first version |
| `carrier_code` | body | integer | yes | Contract/carrier | `1` | Must exist for the quotation |
| `address_sender` | body | object | yes | Sender | - | Can use `address_code` or manual fields |
| `address_receiver` | body | object | yes | Recipient | - | Can use `address_code` or manual fields |
| `payment.type` | body | string | yes | Payment type | `billed` | `billed`, `gateway`, `afterward` |
| `type` | body | string | no | Shipment purpose | `Commercial` | Optional |
| `nfkey` | body | string | no | NF-e key | `12345678901234567890123456789012345678901234` | 44 digits |
| `invoice_number` | body | string | no | Invoice number | `NF-123` | If not sent, uses the internal code |
| `shipment_reference` | body | string | no | Shipment reference | `REF-2024-01` | If not sent, uses the internal ID |
| `attachments` | body | array | no | Attachments | - | PDF, XML, JPEG, PNG |
| `biller` | body | object | no | Biller | - | Used on the invoice |

#### Minimum payload
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj"
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "Doe Jhon",
    "email": "doejhon@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

#### Maximum payload
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "nfkey": "12345678901234567890123456789012345678901234",
  "type": "Commercial",
  "invoice_number": "NF-123",
  "shipment_reference": "REF-2024-01",
  "carrier_code": 1,
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ],
  "address_sender": {
    "address_code": 1
  },
  "address_receiver": {
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Factory LTDA",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj",
    "foreign": false
  },
  "payment": {
    "type": "gateway"
  }
}
```

#### Request example
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "Doe Jhon", "email": "doejhon@example.com", "phone": "11992494597", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

#### Response example
```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

### Retrieve shipment status
- Method and route: `GET /v1/shipping/{shipping}/status`
- `shipping` accepts: quotation UUID, AWB, or 44-digit NF-e key.

### Retrieve status by channel
- Method and route: `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Retrieve documents
- Method and route: `GET /v1/shipping/{shipping}/docs`
- Returns documents, attachments, and, if allowed, embedded docs.

### Retrieve documents by channel
- Method and route: `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Update shipment
- Method and route: `PUT /v1/shipping/{shipping}`
- Use to attach a document and/or link NF-e in an exceptional case.
- Attached files must be in `base64` and accept only `PDF`, `XML`, `JPEG`, or `PNG`.
- The maximum size is `5 MB` per file.

#### Request example
```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "attachments": [
      {
        "description": "Nota Fiscal",
        "base64": "JVBERi0xLjcK..."
      }
    ]
  }'
```

#### Response example
```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Cancel shipment
- Method and route: `DELETE /v1/shipping/{shipping}`
- `reason` is optional in the body and accepts free text.

#### Request example
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Customer requested cancellation"}'
```

#### Response example
```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

### Remove attachments
- Method and route: `DELETE /v1/shipping/{shipping}/docs`
- Send `attachments` with `id` or `name`.

#### Request example
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

#### Response example
```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!"
}
```

## Retrievals and Documents
### Document parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `attachments` | query | boolean | no | Includes attachments | `true` | Default `true` |
| `embedded` | query | boolean | no | Includes embedded docs | `true` | Default `true` |
| `base64` | query | boolean | no | Includes base64 | `true` | Default `true` |
| `documents` | query | string | no | Filters types | `invoice_pdf,awb_pdf` | Comma-separated list |

### Identified document types
- `awb_pdf`
- `awb_zpl`
- `awb_prc_pdf`
- `awb_prc_zpl`
- `awb_prc_fedex_pdf`
- `awb_prc_fedex_zpl`
- `declaration_pdf`
- `invoice_pdf`
- `non_dangerous_pdf`
- `packing_list_pdf`

### Document retrieval example
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `400` | Shipment not found or business rule | Check `quotation`, `carrier_code`, and eligibility |
| `401` | Payment not authorized | Adjust account configuration |
| `404` | Resource not found | Confirm the provided identifier |
| `422` | Validation | Correct payload |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- Shipping v1 is the most complete integration point in the operational cycle.
- See `api-documentos.md` and `api-rastreio.md` for dedicated retrieval and tracking examples.
