# Draft

- [Back to language README](README.md)

## Overview
- Creates, lists, updates, and removes shipping drafts.
- The draft is processed asynchronously and can evolve into quotation + shipment.
- Associated with the main operational flow.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`

## Authentication
- Bearer token in the `Authorization` header.
- Language header: `lang`.

## Conventions
- The draft accepts partial payloads.
- Draft `status` can be `received`, `edited`, `validating`, `incomplete`, `complete`, `processing`, or `error`.

## Usage Flow
1. Send the draft with the data you already have.
2. Wait for asynchronous processing.
3. List or retrieve drafts.
4. Update only drafts in `incomplete` status.
5. Remove drafts when necessary.

## Endpoints
### Create draft
- Method and route: `POST /v1/draft`

### Update draft
- Method and route: `PUT /v1/draft/{code}`
- Only accepts editing an `incomplete` draft.

### List drafts
- Method and route: `GET /v1/draft`
- Can filter by `drafts.drafts_ids`.

### Remove draft
- Method and route: `DELETE /v1/draft`

### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | no | Channel | `1` | - |
| `channel_identifier_number` | body | string | no | Commercial identifier | `334455` | - |
| `carrier_code` | body | integer | no | Preferred carrier | `1` | - |
| `object` | body | string | no | Object | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | no | Type | `Commercial` | - |
| `tax` | body | string | no | Tax | `sender` | - |
| `insurance` | body | boolean | no | Insurance | `true` | - |
| `measurement` | body | string | no | Measurements | `metric` | Default may come from preset |
| `currency_quote` | body | string | no | Quotation currency | `USD` | - |
| `currency_payment` | body | string | no | Payment currency | `BRL` | - |
| `address_sender` | body | object | no | Sender | - | Accepts partial filling |
| `address_receiver` | body | object | no | Recipient | - | Accepts partial filling |
| `boxes` | body | array | no | Boxes | - | Optional |
| `items` | body | array | no | Items | - | Optional |
| `payment.type` | body | string | no | Payment type | `billed` | `billed`, `afterward` |

### Request example
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP", "city": "Diadema"},
    "address_receiver": {"country_code": "US", "city": "Florida"},
    "boxes": [{"height": 10, "width": 10, "depth": 10}],
    "payment": {"type": "billed"}
  }'
```

### Response example
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno"
}
```

## Business Rules
- A draft without a useful payload returns a warning.
- The backend uses user presets when available.
- Drafts can be converted into quotation and shipment through asynchronous processing.

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `400` | Empty, not found, or non-editable draft | Send valid data or choose another draft |
| `422` | Validation | Correct payload |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- The draft is the best option when the integration does not yet have all final data.
- See `api-cotacao-v1.md` and `api-remessa-v1.md` to understand what asynchronous processing will generate.
