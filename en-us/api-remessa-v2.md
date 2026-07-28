# Shipping V2

- [Back to language README](README.md)

## Overview
- Creates shipments with a simplified contract.
- v2 does not expose, in the analyzed source, status, documents, cancellation, or update endpoints.
- Documented version: v2.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Language header: `lang`.

## Conventions
- `Content-Type: application/json`
- `quotation` must have been created in v2.
- `payment.type` still defines the financial flow.

## Usage Flow
1. Create the quotation in v2.
2. Send the shipment with the returned `quotation`.
3. Track the rest of the flow using v1 endpoints, if applicable to the legacy contract.

## Endpoints
### Create shipment
- Method and route: `POST /v2/shipping`
- When to use: new shipping flow.
- Important rules: the quotation must be v2; `payment.type` must be allowed for the user.

#### Differences compared to v1
- v2 keeps the creation endpoint, but the public documentation does not show the other cycle endpoints.
- `attachments` in v2 accepts only `PDF` or `XML` according to the analyzed source.
- The contract is stricter for address field usage.

#### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | yes | Quotation ID | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Must be v2 |
| `carrier_code` | body | integer | yes | Contract | `1` | Must exist |
| `address_sender` | body | object | yes | Sender | - | Accepts manual fields or `address_code` |
| `address_receiver` | body | object | yes | Recipient | - | Accepts manual fields or `address_code` |
| `payment.type` | body | string | yes | Payment type | `billed` | `billed`, `gateway`, `afterward` |
| `attachments` | body | array | no | Attachments | - | PDF/XML in the source |
| `biller` | body | object | no | Biller | - | Optional |
| `nfkey` | body | string | no | NF-e key | - | 44 digits |

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
    "name": "John Doe",
    "email": "johndoe@example.com",
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
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
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
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
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
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
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
  },
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ]
}
```

#### Request example
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

## Minimum and Maximum Payload
- The minimum payload requires only the essential fields to identify the quotation, carrier, addresses, and payment.
- The maximum payload includes NF-e, attachments, biller, and commercial reference.

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `422` | Validation | Adjust contract and required fields |
| `400` | Business rule | Check quotation and payment eligibility |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- The v2 documented in the source is lean; for operational support of documents and tracking, the analyzed source points to v1.
- All quotation and shipping actions in the second version can be tracked through the other first version APIs.
