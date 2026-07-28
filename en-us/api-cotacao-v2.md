# Quotation V2

- [Back to language README](README.md)

## Overview
- Generates quotations with a simplified contract for new integrations.
- Use when you only want to create the quotation in v2.
- Detailed result retrieval does not exist in this version in the analyzed source.
- Documented version: v2.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Language header: `lang: pt|en|es`.

## Conventions
- `Content-Type: application/json`
- `level` exists in the request body/query of creation and controls the calculation response.
- Most common HTTP statuses: `200`, `400`, `422`, `500`.

## Usage Flow
1. Send the quotation in v2.
2. Receive the `quotation` and use this code to create the shipment in v2.
3. There is no quotation retrieval endpoint in v2 in the analyzed source.

## Endpoints
### Create quotation
- Method and route: `POST /v2/quotation`
- When to use: new pricing flow.
- Important rules: `object`, `type`, `tax`, `insurance`, `currency_quote`, `currency_payment`, `address_sender`, `address_receiver`, `boxes`, and/or `items` according to the type.

#### Functional differences compared to v1
- v2 explicitly documents `items` as a quotation type.
- v2 simplifies address fields in the OpenAPI contract, but validation still requires the necessary data according to origin/destination.
- The detailed result is delivered in the `POST` itself.

#### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `object` | body | string | yes | Object shipped | `package` | `doc`, `package`, `pallet`, `not_doc` in the source |
| `type` | body | string | yes | Quotation type | `items` | `simple`, `advanced`, `items` |
| `tax` | body | string | yes | Tax | `sender` | `sender` or `receiver` |
| `insurance` | body | boolean | yes | Insurance | `true` | - |
| `currency_quote` | body | string | yes | Quotation currency | `USD` | ISO 4217 |
| `currency_payment` | body | string | yes | Payment currency | `EUR` | ISO 4217 |
| `address_sender` | body | object | yes | Sender | - | `address_code` or manual |
| `address_receiver` | body | object | yes | Recipient | - | `address_code` or manual |
| `boxes` | body | array | yes | Boxes/pallets | - | Required except when `type=items` |
| `items` | body | array | yes/conditional | Items | - | Rules depend on the type |
| `manufacturer` | body | object | no | Manufacturer | - | Optional |
| `declared_freight` | body | object | no | Declared freight | - | Optional |

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
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
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

#### Request example
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/quotation \
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
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
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

#### Common errors
| Status | Meaning | How to Fix |
|---|---|---|
| `400` | Carrier unavailable or rule denied | Adjust country, weight, box, or contract |
| `422` | Validation | Correct required fields |
| `406` | Incompatible structure | Review payload and version |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- The v2 OpenAPI documentation and request validation are not identical in every detail. Validate with the team responsible for the API.
- For detailed quotation retrieval, the analyzed source does not provide a v2 endpoint equivalent to `GET /v1/quotation/{quotation}/{level}`.
- All quotation and shipping actions in the second version can be tracked through the other first version APIs.
