# Box

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
- Maintains the registry of the packaging you use frequently.
- Once registered, the box can be referenced in the quotation by `box_code` or `box_identifier`,
  without repeating dimensions and weight in each request.
- Registration and removal are **asynchronous**: the API confirms the receipt and processes it in the background.

## Base URL
- Production: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Interactive documentation: `https://api.shipsmart.global/doc` and `https://hmlapi.shipsmart.com.br/doc`

## Authentication
```
Authorization: Bearer YOUR_TOKEN
Content-Type: application/json
lang: pt
```

## Why use the box registry

**Without the registry** — repeating the dimensions in every quotation:
```json
"boxes": [{ "name": "Box S", "height": 20, "width": 20, "depth": 20 }]
```

**With the registry** — referencing the box:
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Advantages: smaller payload, standardized dimensions and use of the `smartbox` grouping
(which fits the items into the registered boxes automatically).

## Usage flow
1. Register the boxes with `POST /v1/box`.
2. Wait a few moments (asynchronous processing).
3. Confirm with `GET /v1/box`.
4. Use `box_code` or `box_identifier` in the [quotations](api-cotacao-v1.md).
5. Remove the obsolete boxes with `DELETE /v1/box`.

## Endpoint: list boxes

- **Method and route:** `GET /v1/box`

| Parameter | Location | Type | Required | Default | Description |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array | no | - | List of `box_code` and/or `box_identifier`. **Empty or absent returns all of them** |
| `page` | query | integer | no | `1` | Listing page |

- **Items per page:** 15
- The listing is **cached for up to 5 minutes**. A newly registered box may take that long to appear.

```bash
# All boxes
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'

# Filtering by identifiers
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Individual cartridge",
        "width": "65",
        "height": "65",
        "depth": "65",
        "weight": "2.8",
        "created_at": "2026-01-27T10:05:05.000000Z",
        "updated_at": "2026-01-27T10:05:05.000000Z",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `box_code` | integer | **Numeric code of the box.** Use it in `boxes[].box_code` in the quotation |
| `box_identifier` | string | **Text identifier of the box.** Use it in `boxes[].box_identifier` in the quotation |
| `name` | string | Box name |
| `height`, `width`, `depth` | number | Registered dimensions |
| `weight` | number | Weight of the empty packaging |
| `active` | boolean | Active box. **Inactive boxes cannot be used in the quotation** |
| `created_at`, `updated_at` | string | Creation and last change dates |

## Endpoint: create or update boxes

- **Method and route:** `POST /v1/box`
- **Behavior:** if the `box_identifier` (or `box_code`) already exists, the box is **updated**.
  Otherwise, it is **created**.
- **Processing:** asynchronous. The `200` response only confirms the receipt.

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `boxes` | array | **yes** | 1 item | - | List of boxes to create/update |
| `boxes[].box_identifier` | string | **yes** | - | 155 | Unique identifier of the box in your base (e.g. `CX0236`) |
| `boxes[].name` | string | **yes** | - | 155 | Descriptive name |
| `boxes[].height` | number | **yes** | 1 | - | Height |
| `boxes[].width` | number | **yes** | 1 | - | Width |
| `boxes[].depth` | number | **yes** | 1 | - | Depth |
| `boxes[].weight` | number | **yes** | - | - | Weight of the empty packaging. Up to 11 integer digits and **3 decimal places** |
| `boxes[].activated` | boolean | **yes** | - | - | `true` to make the box available in the quotations |
| `boxes[].box_code` | string | no | - | 50 | Box code. Inform it to update an existing box by its code |

> The dimensions follow the measurement system of your account/quotation: **centimeters** in `metric`,
> **inches** in `imperial`. The weight is in **kilograms** (`metric`) or **pounds** (`imperial`).

### Minimum payload

```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Individual cartridge",
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
      "box_code": "1",
      "box_identifier": "CX0236",
      "name": "Individual cartridge",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    },
    {
      "box_identifier": "CX0237",
      "name": "Master Box",
      "height": 40,
      "width": 30,
      "depth": 25,
      "weight": 0.85,
      "activated": true
    },
    {
      "box_identifier": "CX0100",
      "name": "Discontinued box",
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 0.2,
      "activated": false
    }
  ]
}
```

### Request example

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "boxes": [
      {"box_identifier": "CX0236", "name": "Individual cartridge", "height": 65, "width": 65, "depth": 65, "weight": 2.8, "activated": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

> **The processing is asynchronous.** Confirm the result with `GET /v1/box` (keeping the 5-minute cache in mind)
> or follow the import report in the panel.

## Endpoint: remove boxes

- **Method and route:** `DELETE /v1/box`
- **Processing:** asynchronous. At the end, a report by e-mail informs how many boxes were removed
  and which ones failed.

| Field | Type | Required | Description |
|---|---|---|---|
| `boxes` | object | **yes** | Envelope |
| `boxes.box_ids` | array | **yes** | List of `box_code` and/or `box_identifier` to remove |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": ["CX0236", 1]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Removal rules

| Rule | Result when it fails |
|---|---|
| The box must exist in your base | Error in the report: `box %s not found!` |
| The box **cannot have linked items** | Error in the report: `box %s cannot be deleted, has related item` |

> To "deactivate" a box that already has linked items, use `POST /v1/box` with `activated: false`
> instead of removing it.

## How to use the boxes in the quotation

| Form | Payload in `boxes[]` | When to use it |
|---|---|---|
| By code | `{ "box_code": 1 }` | You stored the numeric code |
| By identifier | `{ "box_identifier": "CX0236" }` | You use your own packaging code |
| Manual | `{ "height": 20, "width": 20, "depth": 20 }` | One-off packaging, not registered |
| Kit | `{ "kit": "KIT-001" }` | Box **and** items come from the kit registry |

See the `boxes[]` section in [Quotation (v1)](api-cotacao-v1.md#parameters--boxes).

## Business rules
- Registration and removal are processed in an **asynchronous queue**.
- The listing is **cached for 5 minutes** per user and page.
- Boxes with `activated: false` do not appear as available in the quotations
  (`A caixa :name não existe ou está desativada!`).
- Boxes are **per account**: each user sees only their own.
- `box_identifier` must be unique in your base — it is what allows updating instead of duplicating.

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Business rule during the processing | Read `message` and review the payload |
| `403` | No permission for the action | Ask support for the permission |
| `422` | Field validation | Fix the fields listed in `data` (dimensions ≥ 1, `box_identifier` required, `activated` boolean) |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- Registering the boxes is the prerequisite for using the `smartbox` grouping and the `items` quotation type.
- Keep the `box_identifier` values aligned with your ERP/WMS: that greatly simplifies the integration.
- To register the products that go inside the boxes, see [Items](api-itens-v1.md).
