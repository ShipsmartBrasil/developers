# Items

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
- Maintains the registry of the products you ship, with weight, dimensions, HS Code and values.
- With the item registered, the quotation can reference it only by the `sku`, without repeating all the data.
- It is what enables the `type: items` quotation and the automatic `smartbox` grouping.
- All write operations are **asynchronous**.

> ⚠️ **Important:** `POST`, `PUT` and `DELETE` **do not validate the fields synchronously**.
> The API responds `200` confirming the receipt and validates each item during the processing.
> Invalid items are **discarded silently** and the reason is recorded in the import report,
> in the panel. Always confirm the result with `GET /v1/item`.

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

## Why use the item registry

**Without the registry** — repeating everything in each quotation:
```json
"items": [
  { "box_id": 1, "name": "Cotton t-shirt S", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
]
```

**With the registry** — referencing the SKU:
```json
"items": [
  { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
]
```

With `type: items`, you do not even need to indicate the box: the platform chooses among the
[registered boxes](api-caixa-v1.md).

## Usage flow
1. Register the items with `POST /v1/item`.
2. Wait for the asynchronous processing.
3. Confirm with `GET /v1/item`.
4. Use the `sku` in the [quotations](api-cotacao-v1.md).
5. Update with `PUT /v1/item` and remove with `DELETE /v1/item`.

## Endpoint: list items

- **Method and route:** `GET /v1/item`

| Parameter | Location | Type | Required | Default | Description |
|---|---|---|---|---|---|
| `items[skus][]` | query | array | no | - | List of SKUs. **Empty or absent returns all of them** |
| `page` | query | integer | no | `1` | Listing page |

- **Items per page:** 15

```bash
# All items
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'

# Filtering by SKU
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
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
        "name": "Sneaker",
        "description": "Men's running sneaker 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "TSH-000-S",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "unit_value": 220.25,
        "selling_unit_value": 330.00,
        "hscode": "950450",
        "is_groupable": true,
        "is_foldable": false,
        "box_code": 3,
        "box_identifier": "CX0236"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

## Endpoint: create items

- **Method and route:** `POST /v1/item`
- **Processing:** asynchronous, in batch.

### Parameters — `items[]`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `sku` | string | **yes** | - | - | Unique code of the product in your base. **It cannot repeat** |
| `name` | string | **yes** | - | - | Product name (appears on the invoice) |
| `description` | string | **yes** | 6 | 155 | Product description. **Distinct descriptions between items avoid customs alerts** |
| `category` | string | **yes** | 6 | 6 | Product category. **Exactly 6 characters** |
| `country_code` | string | **yes** | 2 | 2 | Country of origin/manufacture in ISO 3166-1 alpha-2 |
| `hscode` | string | **yes** | - | - | NCM/HS Code. Defines the tax table applied at the destination |
| `height` | number | **yes** | 0.01 | - | Product height |
| `width` | number | **yes** | 0.01 | - | Product width |
| `depth` | number | **yes** | 0.01 | - | Product depth |
| `weight` | number | **yes** | 0.01 | - | Unit weight |
| `is_groupable` | boolean | **yes** | - | - | `true` allows grouping the item with others in the same box. `false` forces an exclusive box in the `smartbox` |
| `ean` | string | no | - | - | EAN/GTIN code |
| `unit_value` | number | no | - | - | Default unit value. If absent in the registry **and** in the quotation, the quotation fails |
| `selling_unit_value` | number | no | - | - | Unit selling value. Necessary when the quotation uses `ior: true` |
| `box_code` | integer | no | - | - | Default box of the item ([Box](api-caixa-v1.md)). It must be an **integer** |
| `box_identifier` | string | no | - | - | Default box of the item, by its text identifier |
| `is_foldable` | boolean | no | - | - | Foldable item. In the `smartbox`, foldable ones are fitted by volume; non-foldable ones by dimensions |

> **Dimensions and weight** follow the measurement system of the account/quotation: cm/kg in `metric`,
> in/lb in `imperial`.

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
      "weight": 0.25,
      "is_groupable": true
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
      "description": "100% cotton t-shirt short sleeve white size S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3,
      "box_identifier": "CX0236"
    },
    {
      "sku": "SNK-044",
      "name": "Sport sneaker 44",
      "description": "Men's running sneaker size 44 black",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
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
      {"sku": "TSH-000-S", "name": "Cotton t-shirt S", "description": "100% cotton t-shirt short sleeve white size S", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint: update items

- **Method and route:** `PUT /v1/item`
- **Update key:** the `sku`. Send the existing SKU and the fields to change.
- **Processing:** asynchronous, in batch.
- **Payload structure:** identical to the `POST`.

> ⚠️ **A nonexistent SKU is not created.** The update does not do an "upsert": if the SKU does not exist,
> nothing happens (and nothing is reported in the HTTP response). Use `POST` to create.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Cotton t-shirt S", "description": "Premium 100% cotton t-shirt short sleeve white S", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint: remove items

- **Method and route:** `DELETE /v1/item`
- **Processing:** asynchronous, in batch.

| Field | Type | Required | Description |
|---|---|---|---|
| `items` | object | **yes** | Envelope |
| `items.skus` | array | **yes** | List of SKUs to remove |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Asynchronous validation — reference table

Since the validation happens during the processing, use this table to make sure the items get in.
The first rule violated discards the item.

| Field | Rule | Reason recorded in the report |
|---|---|---|
| (item) | Must be an object | `not_array` |
| `name` | Required, text | `name` |
| `description` | Required, text | `description` |
| `category` | Required, text, **exactly 6 characters** | `category` |
| `country_code` | Required, text, **exactly 2 characters** | `country_code` |
| `sku` | Required, text | `sku` |
| `ean` | If sent, must be text | `ean` |
| `height` | Required, numeric, **≥ 0.01** | `height` |
| `width` | Required, numeric, **≥ 0.01** | `width` |
| `depth` | Required, numeric, **≥ 0.01** | `depth` |
| `weight` | Required, numeric, **≥ 0.01** | `weight` |
| `hscode` | Required, text | `hscode` |
| `is_groupable` | If sent, must be boolean | `is_groupable` |
| `box_code` | If sent, must be an **integer** (not a string) | `box_code` |
| `box_identifier` | If sent, must be text | `box_identifier` |
| `unit_value` | If sent, must be numeric | `unit_value` |
| `selling_unit_value` | If sent, must be numeric | `selling_unit_value` |

Other situations recorded in the report:

| Situation | Behavior |
|---|---|
| `items` empty or not an array | Report: `Itens isn't an array or is empty` |
| SKU duplicated in the registry | The record is rejected (the SKU is a unique index) |
| Nonexistent SKU in the `PUT` | Nothing is changed and nothing is created |
| Invalid SKU in the `DELETE` | Report: `sku_invalid` |

## How to use the items in the quotation

| Quotation `type` | Payload in `items[]` | Behavior |
|---|---|---|
| `advanced` | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | You define which box the item goes into |
| `items` | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | The platform chooses the boxes (account grouping) |
| any | `{ "box_id": 1, "name": "...", "country_code": "BR", "hscode": "...", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | One-off item, not registered |

See the `items[]` section in [Quotation (v1)](api-cotacao-v1.md#parameters--items).

## Business rules
- All write operations run in an **asynchronous queue**.
- The `sku` is a **unique index** per account.
- Items are **per account**: each user sees only their own.
- The registry must be complete for the quotation to work. Related messages:
  - `Nenhum item foi encontrado com o SKU :sku especificado !`
  - `Por favor verifique o cadastro do item :sku, pois poder haver informações imcompletas!`
  - `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).`
  - `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...`
- `is_groupable: false` guarantees an exclusive box in the `smartbox` grouping.
- An incorrect `hscode` generates a tax error in the quotation:
  `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item).`

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `200` with missing items | Item discarded in the asynchronous validation | Check the report in the panel and the validation table above |
| `403` | No permission for the action | Ask support for the permission |
| `422` | `items` absent | Send the `items` array |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- Since the validation is not synchronous, **always confirm with `GET /v1/item`** after registering or updating.
- Keep descriptions distinct between different products: repeated descriptions generate the warning
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` in the quotation.
- To standardize the packaging used with these items, see [Box](api-caixa-v1.md).
