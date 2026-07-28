# Box

- [Back to language README](README.md)

## Overview
- Manages the registration and retrieval of boxes used in quotations, drafts, and shipments.
- Use it to standardize dimensions, weight, and packaging identification.
## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Optional language header: `lang`.

## Conventions
- `Content-Type: application/json`
- Retrieval uses `boxes[box_ids][]` in the query.
- Registration/update occurs asynchronously.

## Usage Flow
1. List the available boxes.
2. Register or update base boxes.
3. Remove boxes when necessary.
4. Reuse the codes in quotations and drafts.

## Endpoints
### List boxes
- Method and route: `GET /v1/box`
- When to use: to fetch registered boxes or filter by IDs.

### Create or update boxes
- Method and route: `POST /v1/box`
- When to use: when maintaining the integrator's packaging base.

### Remove boxes
- Method and route: `DELETE /v1/box`
- When to use: when deleting one or more registered boxes.

## Parameters
### Listing
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `boxes[box_ids][]` | query | array | no | List of codes/identifiers | `1` | Empty returns all |

### Registration
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `boxes[]` | body | array | yes | List of boxes | - | Each item is batch processed |
| `box_code` | body | string | no | Box code | `1` | Optional for update |
| `box_identifier` | body | string | yes | Box identifier | `CX0236` | Required on registration |
| `name` | body | string | yes | Name | `Cartucho Individual` | Up to 155 characters |
| `height` | body | number | yes | Height | `65` | >= 1 |
| `width` | body | number | yes | Width | `65` | >= 1 |
| `depth` | body | number | yes | Depth | `65` | >= 1 |
| `weight` | body | number | yes | Weight | `2.8` | Regex with up to 3 decimal places |
| `activated` | body | boolean | yes | Activated | `true` | - |

### Deletion
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `boxes.box_ids` | body | array | yes | Box IDs | `[1, 2]` | Required list |

## Minimum and Maximum Payload
### Minimum payload
```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

### Maximum payload
```json
{
  "boxes": [
    {
      "box_code": 1,
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

## Request example
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "boxes": [
      {
        "box_identifier": "CX0236",
        "name": "Cartucho Individual",
        "height": 65,
        "width": 65,
        "depth": 65,
        "weight": 2.8,
        "activated": true
      }
    ]
  }'
```

## Response example
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Business Rules
- Registration is processed in an asynchronous job.
- Listing can be cached by user and page.
- Deletion accepts multiple IDs.

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `422` | Validation | Correct required fields |
| `400` | Invalid request | Check payload | 
| `500` | Unexpected error | Contact the responsible team |
