# Manifest

- [Back to language README](README.md)

## Overview
- Runs the synchronous quotation + shipment flow.
- After creation, it may return the operation's final documents.
- Associated with the main operational flow.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Language header: `lang`.

## Conventions
- Optional query `format`: `pdf` or `zpl`.
- The flow uses `billed` as the payment type in the published contract.

## Usage Flow
1. Send the complete manifest data.
2. The API creates the quotation, creates the shipment, and generates documents.
3. If `format` is `pdf` or `zpl`, the response favors that format.

## Endpoints
### Create manifest
- Method and route: `POST /v1/manifest`
- Query params: `format`.

### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `format` | query | string | no | Return type | `pdf` | `pdf` or `zpl` |
| `object` | body | string | yes | Object | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | yes | Purpose | `Commercial` | Comes from the business flow |
| `tax` | body | string | yes | Tax | `sender` | `sender` or `receiver` |
| `insurance` | body | boolean | yes | Insurance | `true` | - |
| `residential_delivery` | body | boolean | yes | Residential delivery | `false` | - |
| `non_stackable` | body | boolean | yes | Non-stackable | `false` | - |
| `measurement` | body | string | yes | Measurement system | `metric` | `metric` or `imperial` |
| `ior` | body | boolean | yes | IOR | `false` | - |
| `currency_quote` | body | string | yes | Quotation currency | `USD` | - |
| `currency_payment` | body | string | yes | Payment currency | `EUR` | - |
| `address_sender` | body | object | yes | Sender | - | Manual or `address_code` |
| `address_receiver` | body | object | yes | Recipient | - | Manual or `address_code` |
| `boxes` | body | array | yes | Boxes | - | Required |
| `items` | body | array | yes | Items | - | Required except `doc` |
| `payment.type` | body | string | yes | Payment | `billed` | Manifest only accepts billed |
| `declared_freight` | body | object | no | Declared freight | - | Optional |

### Minimum payload
```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Florida",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Maximum payload
```json
{
  "format": "pdf",
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "carrier_code": 1,
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Manifesto de teste",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Warehouse",
    "street": "Main Street",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678901",
    "state_tax_id": "1234567890",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida",
    "description": "Office",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "federal_tax_id": "7418529630",
    "state_tax_id": "7418529586",
    "foreign": true
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "Test Product",
      "country_code": "US",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100,
      "selling_unit_value": 110
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Response example
```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "000000",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    }
  }
}
```

## Business Rules
- The manifest only works with billed payment.
- The final document may be PDF, ZPL, or both, depending on the flow and carrier.
- If the quotation is incompatible with the version, execution fails.

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `400` | Carrier error or business rule | Adjust payload or check carrier response |
| `422` | Validation | Correct required fields |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- This endpoint is synchronous and concentrates more responsibility than the separate quotation + shipment flows.
- Recommended only for integrations that really need the full flow in one call.
