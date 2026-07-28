# Items

- [Back to language README](README.md)

## Overview
- Manages registered items for use in quotations, drafts, and shipments.
- Items can be retrieved, created, updated, and removed in batches.
## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Optional language header: `lang`.

## Conventions
- `Content-Type: application/json`
- Retrieval uses `items[skus][]` in the query.
- Registration/update/removal are asynchronous.

## Usage Flow
1. List the items already registered.
2. Register the base items.
3. Update items when there is a change in dimension, weight, or classification.
4. Remove obsolete items.

## Endpoints
### List items
- Method and route: `GET /v1/item`

### Create items
- Method and route: `POST /v1/item`

### Update items
- Method and route: `PUT /v1/item`

### Remove items
- Method and route: `DELETE /v1/item`

## Parameters
### Listing
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `items[skus][]` | query | array | no | List of SKUs | `TSH-000-S` | Empty returns all |

### Registration/update
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `items[]` | body | array | yes | List of items | - | Batch processing |
| `name` | body | string | yes | Item name | `Tenis` | Required |
| `description` | body | string | yes | Description | `Nike 44` | Up to 155 characters |
| `category` | body | string | yes | Category | `950450` | Free value in the source |
| `country_code` | body | string | yes | Country of origin | `BR` | ISO2 |
| `sku` | body | string | yes | SKU | `oLdTsW222` | Required |
| `ean` | body | string | no | EAN | `7894900010015` | Optional |
| `height` | body | integer | yes | Height | `5` | - |
| `width` | body | integer | yes | Width | `5` | - |
| `depth` | body | integer | yes | Depth | `5` | - |
| `weight` | body | number | yes | Weight | `0.3` | - |
| `unit_value` | body | number | no | Unit value | `220.25` | Optional |
| `selling_unit_value` | body | number | no | Sale value | `330.00` | Optional |
| `hscode` | body | string | yes | HS code | `950450` | Required |
| `is_groupable` | body | boolean | yes | Can group | `true` | Required |
| `box_code` | body | integer | no | Default box | `3` | Optional |
| `box_identifier` | body | string | no | Default box | `CX0236` | Optional |

### Deletion
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---|---|---|---|---|
| `items.skus` | body | array | yes | SKUs to remove | `["TSH-000-S"]` | Required |

## Minimum and Maximum Payload
### Minimum payload
```json
{
  "items": [
    {
      "name": "Teste prod 1",
      "description": "Teste prod 1",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "hscode": "673057",
      "is_groupable": false
    }
  ]
}
```

### Maximum payload
```json
{
  "items": [
    {
      "name": "Teste prod 1 update",
      "description": "Teste prod 1 update",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "ean": "7894900010015",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "unit_value": 220.25,
      "selling_unit_value": 330,
      "hscode": "673057",
      "is_groupable": false,
      "box_code": 129,
      "box_identifier": "CX0236"
    }
  ]
}
```

## Request example
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Response example
```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "name": "Tenis",
        "description": "Nike 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "oLdTsW222",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "hscode": "950450",
        "is_groupable": true
      }
    ]
  }
}
```

## Business Rules
- Items are processed in an asynchronous job.
- Listing uses a collection by user.
- `box_code` and `box_identifier` are optional and serve as a default link to a box.

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `422` | Validation | Correct required fields |
| `500` | Unexpected error | Contact the responsible team |
