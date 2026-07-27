# Quote

- [Back to English README](README.md)

## Overview
- Generates international quotes with freight, insurance, and tax breakdowns.
- Use it before creating a shipment.
- The quote is the basis for shipment, tracking, and documents.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Language header accepted: `lang: pt|en|es`.

## Conventions
- `Content-Type: application/json`
- Dates use `Y-m-d H:i:s` in responses.
- Currency codes use ISO 4217.
- `level` controls the response detail: `simple` or `detailed`.
- Common HTTP statuses: `200`, `400`, `404`, `406`, `422`, `500`.

## Usage Flow
1. Send a quote with origin, destination, object type, and boxes/items.
2. Receive the `quotation` code returned by the API.
3. Fetch the result with `GET /v1/quotation/{quotation}/{level}`.
4. If eligible, reuse the same `quotation` to create the shipment.

## Functional Notes
- `simple` calculates by box with direct weight/value.
- `advanced` calculates with boxes and items distributed by `box_id`.
- `items` is accepted by validation, but the public OpenAPI contract does not expose it clearly.

## Eligibility for Shipment
- The quote must exist, not be expired, belong to the same API version, and not have been used yet.
- `package`/`pallet` with `simple` type are not eligible for shipment.
- Shipment also requires a compatible contract and allowed payment type.

## Endpoints
### Create quote
- Method and route: `POST /v1/quotation`
- Important rules: sender and receiver are required; `currency_quote` and `currency_payment` are required.

### Notes
- `object`: `doc`, `package`, `pallet`, `not_doc`
- `type`: `simple`, `advanced`, `items`
- `tax`: `sender` or `receiver`
- `measurement`: `metric` or `imperial`

## Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | no | Sales channel | `1` | Must exist in the registry |
| `channel_identifier_number` | body | string | no | Channel identifier | `334455` | Free text |
| `object` | body | string | yes | Shipment object | `package` | `doc`, `package`, `pallet`, `not_doc` in source |
| `type` | body | string | yes | Quote type | `advanced` | `simple`, `advanced`, `items` in validation |
| `tax` | body | string | yes | Tax payer | `sender` | `sender` or `receiver` |
| `insurance` | body | boolean | yes | Insurance | `true` | - |
| `measurement` | body | string | no | Measurement system | `metric` | `metric` or `imperial` |
| `residential_delivery` | body | boolean | no | Residential delivery | `false` | - |
| `non_stackable` | body | boolean | no | Non-stackable | `false` | - |
| `ior` | body | boolean | no | Importer of Record | `false` | - |
| `currency_quote` | body | string | yes | Quote currency | `BRL` | ISO 4217 |
| `currency_payment` | body | string | yes | Payment currency | `BRL` | ISO 4217 |
| `address_sender` | body | object | yes | Sender | - | Can use `address_code` or manual fields |
| `address_receiver` | body | object | yes | Receiver | - | Can use `address_code` or manual fields |
| `boxes` | body | array | yes | Boxes/packages | - | Required except when `type=items` |
| `items` | body | array | no | Load items | - | Required for `advanced`/`items` |
| `manufacturer` | body | object | no | Invoice manufacturer | - | Optional |
| `declared_freight` | body | object | no | Declared freight | `{"declared":true,"value":0,"currency":"USD"}` | Only when applicable |
| `level` | query | string | no | Detail level | `simple` | `simple` or `detailed` |

## Minimum payload
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

## Maximum payload
```json
{
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
  "residential_delivery": false,
  "non_stackable": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 8, "width": 8, "depth": 8 },
    { "box_code": 3, "height": 10, "width": 10, "depth": 10 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "US",
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

## Example request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/quotation \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: en' \
  --data '{
    "object": "doc",
    "type": "simple",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "BRL",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
    "boxes": [{"height": 10, "width": 10, "depth": 10, "weight": 1, "price": 100}]
  }'
```

## Example response
```json
{
  "status": "success",
  "message": "Quote completed successfully!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2021-09-28 14:28:10",
    "object": "package",
    "type": "advanced",
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "freight": "181.95",
        "tax": "28153.894",
        "currency_payment_amount": "24288.1"
      }
    ]
  }
}
```

## Common Errors
| Status | Meaning | Fix |
|---|---|---|
| `400` | Carrier unavailable or rule denied | Adjust country, weight, box, or contract |
| `422` | Validation | Fix required fields |
| `406` | Incompatible structure | Review payload and version |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- The OpenAPI documentation for v2 and the request validation are not identical in every detail.
- For detailed quote lookup, the analyzed source does not provide a v2 equivalent to `GET /v1/quotation/{quotation}/{level}`.
- All quote and shipment actions in the second version can be tracked through the other APIs in the first version.
