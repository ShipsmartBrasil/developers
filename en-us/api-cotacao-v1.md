# Quotation

- [Back to language README](README.md)

## Overview
- Generates international quotations with freight, insurance, and tax details.
- Use when you need to simulate the shipment before creating the shipment.
- The quotation is the basis for shipping, tracking, and documents.
## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Type: Bearer token.
- Header: `Authorization: Bearer SEU_TOKEN`
- Additional accepted header: `lang: pt|en|es`.
- The default language is `pt`.

## Conventions
- `Content-Type: application/json`
- Dates in `Y-m-d H:i:s` format in responses.
- Currencies in 3-letter ISO 4217 code.
- `level` controls the result detail level: `simple` or `detailed`.
- Common HTTP statuses: `200`, `400`, `404`, `406`, `422`, `500`.

## Usage Flow
1. Send a quotation with origin, destination, object, type, and items/boxes.
2. Receive the `quotation` returned by the API.
3. Retrieve the result with `GET /v1/quotation/{quotation}/{level}`.
4. If the quotation is eligible, use the same `quotation` to create the shipment.

## Contract Differences
| Endpoint | Main difference | Impact |
|---|---|---|
| `POST /v1/quotation` | Creates the quotation | Initial flow |
| `GET /v1/quotation/{quotation}/{level}` | Retrieves result | Allows reviewing carriers |

## Functional Topics
### Simple, advanced, and `items` quotation
- `simple`: calculates by box with direct weight/value. For `doc`, it is the natural flow.
- `advanced`: calculates with boxes and items distributed by `box_id`.
- `items`: backend validation accepts this type, but the OpenAPI contract does not expose it clearly. Validate with the team responsible for the API.

### `package`, `pallet`, and `doc`
- `doc`: document. Simple quotation for documents uses a box with weight and price.
- `package`: package/box.
- `pallet`: pallet. Can use `volume` instead of `height/width/depth`.

### Quotation eligible for shipping
- The quotation must exist, not be expired, belong to the same shipment `api_version`, and not have been used yet.
- `package`/`pallet` with `simple` type are not eligible for shipping.
- The shipment also requires a compatible contract and allowed payment type.

## Endpoints
### Create quotation
- Objective: calculate freight, insurance, and taxes.
- Method and route: `POST /v1/quotation`
- When to use: before creating a shipment.
- Important rules: origin and destination are required; `currency_quote` and `currency_payment` are required; `measurement` accepts `metric` or `imperial`.

#### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | no | Sales channel | `1` | Must exist in registration |
| `channel_identifier_number` | body | string | no | Channel identifier | `334455` | Free text |
| `object` | body | string | yes | Object shipped | `package` | `doc`, `package`, `pallet`, `not_doc` in the source |
| `type` | body | string | yes | Quotation type | `advanced` | `simple`, `advanced`, `items` in validation |
| `tax` | body | string | yes | Who pays taxes | `sender` | `sender` or `receiver` |
| `insurance` | body | boolean | yes | Insurance | `true` | - |
| `measurement` | body | string | no | Measurement system | `metric` | `metric` or `imperial` |
| `residential_delivery` | body | boolean | no | Residential delivery | `false` | - |
| `non_stackable` | body | boolean | no | Indicates non-stackable | `false` | - |
| `ior` | body | boolean | no | Importer of Record | `false` | - |
| `currency_quote` | body | string | yes | Quotation currency | `BRL` | ISO 4217 |
| `currency_payment` | body | string | yes | Payment currency | `BRL` | ISO 4217 |
| `address_sender` | body | object | yes | Origin | - | Can use `address_code` or manual fields |
| `address_receiver` | body | object | yes | Destination | - | Can use `address_code` or manual fields |
| `boxes` | body | array | yes | Boxes/packages | - | Required except when `type=items` |
| `items` | body | array | no | Cargo items | - | Required in `advanced`/`items` |
| `manufacturer` | body | object | no | Manufacturer on the invoice | - | Optional |
| `declared_freight` | body | object | no | Declared freight | `{"declared":true,"value":0,"currency":"USD"}` | Only when applicable |
| `level` | query | string | no | Detail level | `simple` | `simple` or `detailed` |

#### Minimum payload
```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida"
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ]
}
```

#### Maximum payload
```json
{
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": false,
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida"
  },
  "boxes": [
    {
      "name": "Caixa 1",
      "height": 20,
      "width": 20,
      "depth": 20
    }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "BR",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100
    }
  ],
  "manufacturer": {
    "name": "Factory LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj"
  }
}
```

#### Request example
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/quotation \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "simple",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "BRL",
    "currency_payment": "BRL",
    "address_sender": {
      "country_code": "BR",
      "state_code": "SP",
      "postal_code": "09941070",
      "city": "Diadema"
    },
    "address_receiver": {
      "country_code": "US",
      "state_code": "FL",
      "postal_code": "33101",
      "city": "Florida"
    },
    "boxes": [{"height": 10, "width": 10, "depth": 10, "weight": 1, "price": 100}]
  }'
```

#### Response example
```json
{
  "status": "success",
  "message": "Cotação Realizada com sucesso!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2021-09-28 14:28:10",
    "description": null,
    "channel": "Domenic Dooley",
    "channel_identifier_number": "334455",
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "valid_until": "2021-09-28 20:28:10",
        "transit_days": 5,
        "freight": "181.95",
        "insurance": "11.45",
        "tax": "28153.894",
        "currency_payment_amount": "24288.1"
      }
    ]
  }
}
```

#### Common errors
| Status | Meaning | How to Fix |
|---|---|---|
| `400` | Business rule failure | Adjust weight, postal code, eligibility, or quotation type |
| `406` | Inconsistent payload | Check required fields and types |
| `422` | Validation | Correct required fields/formats |
| `500` | Unexpected error | Retry and contact the responsible team |

## Retrieve quotation
### Method and route
- `GET /v1/quotation/{quotation}/{level}`
- `level` required in the path: `simple` or `detailed`.

### Example
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Summary response
```json
{
  "status": "success",
  "message": "Resultado da cotação solicitada!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "object": "package",
    "type": "advanced",
    "carriers": []
  }
}
```

## Final Notes
- The response may include `warnings` when there are registration or transport alerts.
- The analyzed source shows divergence between OpenAPI documentation and validation in some fields. Validate with the team responsible for the API before integrating in production.
