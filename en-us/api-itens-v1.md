# Items

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Maintains the registry of the products you ship, with weight, dimensions, HS Code, values and identifiers (SKU, EAN and NSMP).
- With the item registered, the quotation can reference it only by the `sku`, without repeating all the data.
- It is what enables the [quotation by items](api-cotacao-v2.md#items-quotation) (`type: items`, **exclusive to v2**) and the automatic box groupings of the account.
- All write operations (`POST`, `PUT`, `DELETE`) are **asynchronous** and validated only during processing.
- The listing is **paginated (20 per page)** and reflects the registry in real time (no cache).

> ⚠️ **Important:** `POST`, `PUT` and `DELETE` **do not validate the fields synchronously**.
> The API responds `200` confirming receipt and validates each item during processing.
> Invalid items are **silently discarded** (no error in the HTTP response) and the reason is recorded in the
> item import report, in the dashboard. Always confirm the result with `GET /v1/item`.

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

| Header | Required | Values | Default |
|---|---|---|---|
| `Authorization` | **yes** | `Bearer YOUR_TOKEN` | - |
| `Content-Type` | yes in `POST`, `PUT` and `DELETE` | `application/json` | - |
| `lang` | no | `pt`, `en`, `es` | `pt` |

Each operation requires a specific permission in your profile (list, create/update and remove items).
Without the permission, the response is `403` with `Has no permission!`.

## Usage flow
1. (Optional) Register the [boxes](api-caixa-v1.md) and note the `box_code` to link to the items.
2. Register the items with `POST /v1/item`.
3. Wait for the asynchronous processing and check the import report in the dashboard.
4. Confirm with `GET /v1/item` that all SKUs were added.
5. Use the `sku` in the [quotations](api-cotacao-v1.md).
6. Update with `PUT /v1/item` and remove with `DELETE /v1/item`.

## Key concepts

### Why use the item registry

**Without registration** — repeating everything in each quotation:
```json
"items": [
  { "box_id": 1, "name": "Cotton t-shirt S", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
]
```

**With registration** — referencing the SKU:
```json
"items": [
  { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
]
```

With `type: items` (**exclusive to [Quotation v2](api-cotacao-v2.md#items-quotation)**), you do not even need to place the item in a box: the platform builds the boxes according to the
grouping configured in the account (see [How to use the items in the quotation](#how-to-use-the-items-in-the-quotation)).

### Product identifiers

| Field | What it is | Use |
|---|---|---|
| `sku` | Your internal product code | **Registry key**: unique per account, used to reference, update and remove |
| `ean` | EAN/GTIN barcode | Printed on the invoice and sent as the standard manufacturer identifier in the regulatory information |
| `nsmp` | *Non Standard Manufacturer Product Id* — **non-standardized** manufacturer product identifier (e.g., the manufacturer part/model code) | Printed on the invoice (`Mfr:` line) and sent in the regulatory information required by some carriers for destinations in the **European Union** |

### Units of measurement

Registry dimensions and weight are always interpreted in **centimeters** and **kilograms**.
The `imperial` system conversion performed in the quotation applies only to the measurements sent in the quotation payload —
it is **not** applied to the data filled in from the registry. Always register in cm/kg.

### Data types (processing is strict)

The asynchronous validation checks the **JSON type** of each field. Values with the wrong type discard the item:

| Field | Send as | Correct example | Discarded example |
|---|---|---|---|
| `sku`, `hscode`, `ean`, `nsmp`, `category` | text (in quotes) | `"hscode": "610910"` | `"hscode": 610910` |
| `box_code` | integer | `"box_code": 3` | `"box_code": "3"` or `3.0` |
| `height`, `width`, `depth`, `weight`, `unit_value`, `selling_unit_value` | number | `"weight": 0.25` | `"weight": "0,25"` |
| `is_groupable`, `is_foldable` | JSON boolean | `"is_groupable": false` | `"is_groupable": "false"` (it is stored, but treated as `true` in the quotation) |

## Endpoint: list items

- **Method and route:** `GET /v1/item`
- **When to use:** to query the registry, check whether the asynchronous import has finished and verify the data of a SKU.

### Parameters

| Parameter | Location | Type | Required | Default | Description |
|---|---|---|---|---|---|
| `items[skus][]` | query | array of string | no | - | List of SKUs (exact match, case-sensitive). **Empty or absent returns all** |
| `page` | query | integer | no | `1` | Listing page |

- **Items per page:** 20 (fixed).
- Nonexistent SKUs are ignored (they do not generate an error).
- No cache: processed changes appear immediately.

### Minimum payload
No parameters (returns all items, page 1):
```
GET /v1/item
```

### Maximum payload
```
GET /v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044&page=1
```

### Request example

```bash
# All items
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'

# Filtering by SKU
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

> The `--globoff` option prevents curl from interpreting the brackets in the query string.

### Response example

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "sku": "TSH-000-S",
        "name": "Cotton t-shirt S",
        "description": "100% cotton t-shirt short sleeve white size S",
        "category": "610910",
        "country_code": "BR",
        "hscode": "610910",
        "ean": "7894900010015",
        "nsmp": "FAB-TSH-P-001",
        "height": 2,
        "width": 20,
        "depth": 25,
        "weight": 0.25,
        "is_groupable": true,
        "is_foldable": true,
        "box_code": 3,
        "box_identifier": "CX0236",
        "unit_value": 29.9,
        "selling_unit_value": 49.9
      },
      {
        "sku": "SNK-044",
        "name": "Sport sneaker 44",
        "description": "Mens running sneaker size 44 black",
        "category": "640411",
        "country_code": "BR",
        "hscode": "640411",
        "height": 15,
        "width": 22,
        "depth": 33,
        "weight": 0.8,
        "is_groupable": false,
        "nsmp": null,
        "unit_value": null,
        "selling_unit_value": null
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/item?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 20,
    "prev_page_url": null,
    "to": 2,
    "total": 2
  }
}
```

### Response fields

**Item (`data.data[]`):**

| Field | Type | Always present | Description |
|---|---|---|---|
| `sku` | string | yes | Unique product code |
| `name` | string | yes | Product name |
| `description` | string | yes | Product description |
| `category` | string | yes | Category (6 characters) |
| `country_code` | string | yes | Country of origin/manufacture |
| `hscode` | string | yes | NCM/HS Code |
| `height`, `width`, `depth` | number | yes | Dimensions (cm) |
| `weight` | number | yes | Unit weight (kg) |
| `nsmp` | string \| null | **yes** | Non-standardized manufacturer identifier. Comes as `null` when not registered |
| `unit_value` | number \| null | **yes** | Default unit value. `null` when not registered |
| `selling_unit_value` | number \| null | **yes** | Unit selling value. `null` when not registered |
| `ean` | string | no | EAN/GTIN code (only appears if registered) |
| `is_groupable` | boolean | no | Allows grouping with other items (only appears if registered) |
| `is_foldable` | boolean | no | Foldable item (only appears if registered) |
| `box_code` | integer | no | Linked default box (only appears if registered) |
| `box_identifier` | string | no | Linked default box, by identifier (only appears if registered) |

> The fields are returned as they were stored (same type and value sent). The order of the keys may vary.
> Fields outside the documented list that are sent in the registration are also stored and may appear in the listing:
> **send only the documented fields**.

**Pagination (`data`):** same fields as the [box listing](api-caixa-v1.md#response-fields)
(`current_page`, `from`, `to`, `per_page` = 20, `total`, `last_page`, navigation URLs, `path` and `links`).

## Endpoint: create items

- **Method and route:** `POST /v1/item`
- **When to use:** to register new products, one or several per request.
- **Processing:** asynchronous, in batch. Each item is validated individually; the valid ones are stored and the invalid ones
  are discarded and listed in the import report.
- **An existing SKU is not updated:** use `PUT` to change it.

### Parameters

**Payload root:**

| Field | Type | Required | Description |
|---|---|---|---|
| `items` | array | **yes** | List of items. Without `items`, nothing is stored (the response is still `200`) |

**Parameters — `items[]`:**

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `sku` | string | **yes** | 1 | 50 (recommended) | - | Unique product code in your account. **Cannot be repeated.** The quotation accepts SKUs of up to 50 characters |
| `name` | string | **yes** | 1 | - | - | Product name (appears on the invoice) |
| `description` | string | **yes** | 1 (6 recommended) | 155 (recommended) | - | Product description. **Distinct descriptions across items avoid customs alerts** |
| `category` | string | **yes** | 6 | 6 | - | Product category. **Exactly 6 characters** (no accents: accented letters count as 2) |
| `country_code` | string | **yes** | 2 | 2 | - | Country of origin/manufacture in ISO 3166-1 alpha-2, uppercase (e.g., `BR`) |
| `hscode` | string | **yes** | 1 | - | - | NCM/HS Code, **as text**. Defines the tax table applied at the destination |
| `height` | number | **yes** | 0.01 | - | - | Product height (cm) |
| `width` | number | **yes** | 0.01 | - | - | Product width (cm) |
| `depth` | number | **yes** | 0.01 | - | - | Product depth (cm) |
| `weight` | number | **yes** | 0.01 | - | - | Unit weight (kg) |
| `is_groupable` | boolean | no (**recommended**) | - | - | `true` | `true` allows grouping the item with others in the same box; `false` requires an exclusive box. In the `default` grouping, an item without this field causes an error in the quotation |
| `ean` | string | no | - | 50 (recommended) | - | EAN/GTIN code, **as text** |
| `nsmp` | string | no | - | 50 (recommended) | `null` | *Non Standard Manufacturer Product Id*: non-standardized manufacturer identifier (e.g., model/part code). Printed on the invoice and used in the regulatory information of some carriers for destinations in the European Union. The quotation accepts up to 50 characters |
| `unit_value` | number | no | - | - | - | Default unit value. If absent in the registry **and** in the quotation, the quotation fails |
| `selling_unit_value` | number | no | - | - | - | Unit selling value. Required when the quotation uses `ior: true` |
| `box_code` | integer | no | - | - | - | Default box of the item ([Box](api-caixa-v1.md)). It must be a JSON **integer** |
| `box_identifier` | string | no | - | 50 | - | Default box of the item, by text identifier |
| `is_foldable` | boolean | no | - | - | `false` | Foldable item. In `smartbox`, foldable items are fitted by volume; non-foldable ones, by dimensions |

> **Default box (Form A / Form B):** link the box by `box_code` (Form A) **or** by `box_identifier` (Form B).
> Do not send both with values of different boxes. The existence of the box is **not** checked in the item registration —
> it is only verified in the quotation. Prefer `box_code`: it is the link recognized by the box removal protection.

> `null` values in `unit_value` and `selling_unit_value` are ignored (the field is not stored).

### Minimum payload

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Cotton t-shirt S",
      "description": "100% cotton t-shirt short sleeve white size S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25
    }
  ]
}
```

> This is the smallest accepted payload. To use the item in the `type: items` quotation with the `default` grouping, also include
> `is_groupable` and the default box (`box_code` or `box_identifier`).

### Maximum payload

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Cotton t-shirt S",
      "description": "100% cotton t-shirt short sleeve white size S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "nsmp": "FAB-TSH-P-001",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Sport sneaker 44",
      "description": "Mens running sneaker size 44 black",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": "FAB-SNK-44-BLK",
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 199.9,
      "selling_unit_value": 349.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

### Request example

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Cotton t-shirt S", "description": "100% cotton t-shirt short sleeve white size S", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-001", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true, "box_code": 3}
    ]
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success` (received) or `error` (unexpected failure) |
| `message` | string | Message translated according to the `lang` header |

> The `200` only confirms receipt. Discarded items appear in the import report (see
> [Asynchronous validation](#asynchronous-validation--reference-table)).

## Endpoint: update items

- **Method and route:** `PUT /v1/item`
- **When to use:** to change data of already registered products, in batch.
- **Update key:** the `sku`. The item registered with that SKU receives the values sent.
- **Processing:** asynchronous, in batch, with **the same validation as the `POST`**.

> ⚠️ **Send the complete item.** The `PUT` validation is identical to the `POST` one: `sku`, `name`, `description`, `category`,
> `country_code`, `hscode`, `height`, `width`, `depth` and `weight` are **required in every update**. An item
> sent only with the SKU and the changed field is **discarded** (reason `name` in the report).

> ⚠️ **A nonexistent SKU is not created.** The update does not perform an "upsert": if the SKU does not exist, nothing happens
> (and nothing is reported). Use `POST` to create.

Update rules for optional fields:

| Situation in the `PUT` payload | Result in the registry |
|---|---|
| Optional field **omitted** | Keeps the current value |
| Optional field sent with a value | Replaces the current value |
| `ean`, `nsmp`, `box_code`, `box_identifier`, `is_foldable` sent as `null` | The value is erased (becomes `null`) |
| `unit_value` or `selling_unit_value` sent as `null` | **Ignored**: the current value is kept (it is not possible to erase the price via `PUT`; send a new value) |

### Parameters

Same structure and same rules as [`POST /v1/item`](#endpoint-create-items): root `items` (array) and the `items[]` fields.

### Minimum payload

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Cotton t-shirt S",
      "description": "Premium 100% cotton t-shirt short sleeve white S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27
    }
  ]
}
```

### Maximum payload

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Cotton t-shirt S",
      "description": "Premium 100% cotton t-shirt short sleeve white S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "nsmp": "FAB-TSH-P-002",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27,
      "unit_value": 34.9,
      "selling_unit_value": 54.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Sport sneaker 44",
      "description": "Mens running sneaker size 44 black",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": null,
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 189.9,
      "selling_unit_value": 339.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

> In the second item, `"nsmp": null` erases the registered NSMP.

### Request example

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Cotton t-shirt S", "description": "Premium 100% cotton t-shirt short sleeve white S", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-002", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success` (received) or `error` (unexpected failure) |
| `message` | string | Message translated according to the `lang` header |

## Endpoint: remove items

- **Method and route:** `DELETE /v1/item`
- **When to use:** to permanently delete products from the registry.
- **Processing:** asynchronous, in batch. Nonexistent SKUs are ignored without warning.

### Parameters

| Field | Location | Type | Required | Min. | Description |
|---|---|---|---|---|---|
| `items` | body | object | **yes** | - | Envelope |
| `items.skus` | body | array of string | **yes** | 1 item | List of SKUs to remove. Each SKU must be non-empty text |

> The absence of `items.skus` does not generate an error in the HTTP response (`200`); the problem is recorded in the report.

### Minimum payload

```json
{
  "items": {
    "skus": ["TSH-000-S"]
  }
}
```

### Maximum payload

```json
{
  "items": {
    "skus": ["TSH-000-S", "TSH-000-M", "SNK-044"]
  }
}
```

### Request example

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

### Response example

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success` (received) or `error` (unexpected failure) |
| `message` | string | Message translated according to the `lang` header |

## Asynchronous validation — reference table

Since validation happens during processing (`POST` and `PUT`), use this table to ensure the items are accepted.
The rules are checked **in this order** and the **first** rule violated discards the item.
In the import report, each discarded item appears as:

```
sku => TSH-000-S, status => invalid, reason => category
```

| Order | Field | Rule | `reason` in the report |
|---|---|---|---|
| 1 | (item) | Must be an object | `not_array` |
| 2 | `name` | Required, non-empty text | `name` |
| 3 | `description` | Required, non-empty text | `description` |
| 4 | `category` | Required, text, **exactly 6 characters** | `category` |
| 5 | `country_code` | Required, text, **exactly 2 characters** | `country_code` |
| 6 | `sku` | Required, non-empty text | `sku` |
| 7 | `ean` | If sent (not null), must be text | `ean` |
| 8 | `nsmp` | If sent (not null), must be text | `nsmp` |
| 9 | `height` | Required, numeric, **≥ 0.01** | `height` |
| 10 | `width` | Required, numeric, **≥ 0.01** | `width` |
| 11 | `depth` | Required, numeric, **≥ 0.01** | `depth` |
| 12 | `weight` | Required, numeric, **≥ 0.01** | `weight` |
| 13 | `hscode` | Required, non-empty text | `hscode` |
| 14 | `box_code` | If sent (not null), must be an **integer** (not a string, not a decimal) | `box_code` |
| 15 | `box_identifier` | If sent (not null), must be text | `box_identifier` |
| 16 | `unit_value` | If sent (not null), must be numeric | `unit_value` |
| 17 | `selling_unit_value` | If sent (not null), must be numeric | `selling_unit_value` |

Rules applied silently (no discard and no warning):

| Situation | Behavior |
|---|---|
| `is_groupable` with any value | Accepted and stored as sent. Use a JSON boolean: `"false"` as text is treated as `true` in the quotation |
| `is_foldable` with any value | Not validated; stored as sent |
| `name`, `description`, `sku`, `ean`, `nsmp` above the recommended limits | Stored, but may generate an error when the item is used in the quotation/shipment |
| `country_code` with 2 characters that is not a valid country | Stored; the error appears only in the quotation |
| `box_code`/`box_identifier` of a nonexistent box | Stored; the error appears only in the quotation |
| Undocumented fields | Stored as sent |

Other situations recorded in the report:

| Operation | Situation | Behavior |
|---|---|---|
| `POST`/`PUT` | Empty body (`{}`) | Report: `Itens isn't an array or is empty` |
| `POST`/`PUT` | `items` absent with other fields in the body, or `items` is not a list | Processing fails and is recorded as a failure in the report; nothing is stored |
| `POST`/`PUT` | `items: []` | Nothing is stored and nothing is reported |
| `POST` | SKU already registered, or repeated in the same batch | The duplicate item is rejected; the other valid items of the batch are stored and the batch is marked as failed in the report |
| `PUT` | Nonexistent SKU | Nothing is changed, nothing is created and nothing is reported |
| `DELETE` | `items.skus` absent or empty | Report: `SKUs isn't an array or is empty` |
| `DELETE` | Empty SKU or SKU that is not text (e.g., a number) | Report: `sku => ..., status => invalid, reason => sku_invalid` |
| `DELETE` | Nonexistent SKU | Ignored without warning |

## How to use the items in the quotation

| Quotation `type` | Payload in `items[]` | Behavior |
|---|---|---|
| `advanced` (v1 and v2) | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | You define which box (position in `boxes[]`) the item goes into |
| `items` (**v2 only**) | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | The platform builds the boxes according to the account grouping |
| any | `{ "box_id": 1, "name": "Cotton t-shirt S", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | One-off item, not registered |

When the quotation item has a `sku` and some data is missing (name, country, HS Code, dimensions, weight, values or `nsmp`),
the missing data is completed from the registry. The fields sent in the quotation take priority over the registry.

Use of the registry according to the account grouping (`type: items` quotation, v2):

| Grouping | Registry requirements | `is_groupable` |
|---|---|---|
| `default` | Each item must have a default box (`box_code` or `box_identifier`) and `is_groupable` registered | `true`: grouped with other items of the same default box. `false`: one exclusive box per unit |
| `smartbox` | The candidate boxes come from `boxes[]` in the quotation payload | Absent = `true`. `false`: exclusive box. `is_foldable` (absent = `false`) defines fitting by volume or by dimensions |
| `all_in_one` | Only the item data | Absent = `true`. `false`: the item is left out of the consolidation, in its own box |

See the `items[]` section in [Quotation (v1)](api-cotacao-v1.md#parameters--items).

## Business rules
- All write operations run in the **background**; the `200` only confirms receipt.
- The `sku` is **unique** per account: the `POST` does not overwrite an existing SKU.
- Items are **per account**: each user sees and changes only their own.
- The `PUT` requires the complete item (same validation as the `POST`) and does not create new SKUs.
- Registry dimensions and weight are always in **cm** and **kg**.
- `nsmp` is optional; when filled in, it is printed on the invoice and sent in the regulatory information of some
  carriers for destinations in the European Union. Without `nsmp`, this data remains empty.
- The registry must be complete for the quotation to work. Related messages in the quotation:

| Status | Message | Cause |
|---|---|---|
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | SKU not registered and item without the minimum data in the payload |
| `404` | `O item com o sku :name não existe!` | SKU not registered in the `type: items` quotation |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | Item without a valid default box in the `default` grouping |
| `400` | `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).` | No `unit_value` in the registry or in the quotation |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | Zero value |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | Incorrect `hscode` for the destination |

> The HTTP codes of the quotation messages above follow the [Quotation (v1)](api-cotacao-v1.md#common-errors) page.

## Common errors

| Status | Meaning/Message | How to fix |
|---|---|---|
| `200` with missing items | Item discarded in the asynchronous validation (`reason => field` in the report) | Check the report in the dashboard and the [validation table](#asynchronous-validation--reference-table) |
| `200` with no effect on `PUT` | Nonexistent SKU or incomplete item | Create with `POST` or send the complete item |
| `403` | `Has no permission!` — profile without permission to list, create/update or remove items | Ask support for the permission |
| `422` | `O campo items.skus deve conter um array.` (`GET` only) | Send `items[skus][]` as a list in the query string |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Retry with increasing intervals; if it persists, contact support |
| Report | `Itens isn't an array or is empty` | Send `items` with at least one item |
| Report | `SKUs isn't an array or is empty` | In the `DELETE`, send `items.skus` with at least one SKU |
| Report | `sku => ..., status => invalid, reason => sku_invalid` | In the `DELETE`, send SKUs as non-empty text |

> `POST`, `PUT` and `DELETE` **do not return `422`**: field problems appear only in the import report.

## Final notes
- Since validation is not synchronous, **always confirm with `GET /v1/item`** after registering, updating or removing.
- Send the correct JSON types: `sku`, `hscode`, `ean` and `nsmp` as text; `box_code` as an integer; booleans without quotes.
- Keep descriptions distinct across different products: repeated descriptions generate the warning
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` in the quotation.
- Fill in `nsmp` for products shipped to the European Union whenever the manufacturer has its own product code.
- To standardize the packaging used with these items, see [Box](api-caixa-v1.md).
- To use the items in the quotation, see [Quotation (v1)](api-cotacao-v1.md).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
