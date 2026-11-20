# Box

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Maintains the registry of the packaging you use frequently (name, dimensions and weight of the empty packaging).
- Once registered, the box can be referenced in the quotation by `box_code` or `box_identifier`,
  without repeating dimensions and weight in each request.
- The box can also be linked to a [registered item](api-itens-v1.md) as the default box of the product.
- Creation/update and removal are **asynchronous**: the API validates the format, confirms receipt and processes in the background.
- The listing is **paginated (15 per page)** and cached for up to **5 minutes**.

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
| `Content-Type` | yes in `POST` and `DELETE` | `application/json` | - |
| `lang` | no | `pt`, `en`, `es` | `pt` |

Each operation requires a specific permission in your profile (list, create and remove boxes).
Without the permission, the response is `403` with `Has no permission!`.

## Usage flow
1. Register the boxes with `POST /v1/box`.
2. Wait a few moments (asynchronous processing). Follow the result through the import report in the dashboard.
3. Confirm with `GET /v1/box` and note the generated `box_code` (remember the cache of up to 5 minutes).
4. Use `box_code` or `box_identifier` in the [quotations](api-cotacao-v1.md) and in the [items](api-itens-v1.md) registry.
5. Remove obsolete boxes with `DELETE /v1/box` (preferably by `box_code`).

## Key concepts

### Why use the box registry

**Without registration** — repeating the dimensions in every quotation:
```json
"boxes": [{ "name": "Box S", "height": 20, "width": 20, "depth": 20, "weight": 0.3 }]
```

**With registration** — referencing the box:
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Advantages: smaller payload, standardized dimensions and the possibility of linking each registered item to a
default box (used in the `default` grouping of the `type: items` quotation, exclusive to v2).

### `box_code` x `box_identifier`

| Field | Who defines it | Type in the listing | Use |
|---|---|---|---|
| `box_code` | **The platform** (generated at registration) | integer | Numeric code of the box. The most reliable way to reference the box in quotation, item and removal |
| `box_identifier` | **You** (in the `POST`) | string | Your own packaging code (e.g., `CX0236`). It must be unique in your account |

> **Avoid a `box_identifier` made up only of numbers** (e.g., `"123"`). In the listing filters and in removal,
> any numeric value is interpreted as `box_code`. Prefer identifiers with letters, such as `CX0123`.

### Units of measurement

The dimensions and weight of registered boxes are always interpreted in **centimeters** and **kilograms**.
The `imperial` system conversion (inches/pounds) performed in the quotation applies only to the measurements sent
in the quotation payload itself — it is **not** applied to the data coming from the registry. Always register in cm/kg.

## Endpoint: list boxes

- **Method and route:** `GET /v1/box`
- **When to use:** to query the registered boxes, find out the generated `box_code` and check whether the asynchronous registration has finished.

### Parameters

| Parameter | Location | Type | Required | Default | Description |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array of string/integer | no | - | List of `box_code` and/or `box_identifier` (can be mixed). **Empty or absent returns all** |
| `page` | query | integer | no | `1` | Listing page |

- **Items per page:** 15 (fixed).
- Numeric values in `boxes[box_ids][]` are searched as `box_code`; the others, as `box_identifier`.
- Nonexistent identifiers are simply ignored (they do not generate an error).

> ⚠️ **5-minute cache:** the result is cached for up to 5 minutes for each combination of account and filter.
> During this interval:
> - newly registered, changed or removed boxes may not be reflected in the listing;
> - **requests for different pages with the same filter may return the same page already queried**
>   (the page number does not differentiate the cache). If you have more than 15 boxes, prefer querying by
>   `boxes[box_ids][]` or wait for the cache to expire between one page and the next.

### Minimum payload
No parameters (returns all boxes, page 1):
```
GET /v1/box
```

### Maximum payload
```
GET /v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1&page=1
```

### Request example

```bash
# All boxes
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'

# Filtering by identifier and by code
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
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
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Individual cartridge",
        "height": 65,
        "width": 65,
        "depth": 65,
        "weight": 2.8,
        "created_at": "2026-01-27 10:05:05",
        "updated_at": "2026-01-27 10:05:05",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/box?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Response fields

**Box (`data.data[]`):**

| Field | Type | Description |
|---|---|---|
| `box_code` | integer | **Numeric code of the box**, generated by the platform. Use it in `boxes[].box_code` in the quotation and in the item `box_code` |
| `box_identifier` | string \| null | **Your text identifier of the box**. Use it in `boxes[].box_identifier` in the quotation. It may come as `null` for boxes created without an identifier through the dashboard |
| `name` | string | Box name |
| `height`, `width`, `depth` | number | Registered dimensions (cm) |
| `weight` | number | Weight of the empty packaging (kg) |
| `created_at`, `updated_at` | string | Creation and last change dates, in the `AAAA-MM-DD HH:MM:SS` (YYYY-MM-DD HH:MM:SS) format |
| `active` | boolean | Indicates whether the box is active. Also treat `1`/`0` as true/false |

**Pagination (`data`):**

| Field | Type | Description |
|---|---|---|
| `current_page` | integer | Current page |
| `from`, `to` | integer \| null | Position of the first and last record of the page |
| `per_page` | integer | Records per page (15) |
| `total` | integer | Total records found |
| `last_page` | integer | Number of the last page |
| `first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url` | string \| null | Navigation URLs (they keep the query filters) |
| `path` | string | Base URL of the listing |
| `links` | array | Ready-made navigation links (`url`, `label`, `active`) |

## Endpoint: create or update boxes

- **Method and route:** `POST /v1/box`
- **When to use:** to register new boxes or change existing boxes, in batch.
- **Behavior:** to **update** an existing box, inform its `box_identifier` (or its `box_code`).
  If the box does not exist, it is **created**.
- **Processing:** the API validates the format of all fields (error `422` if something is wrong) and, if everything is fine,
  responds `200` confirming **only receipt**. Storage happens in the background and the result is available in the
  box import report, in the dashboard.

### Parameters

**Payload root:**

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `boxes` | array | **yes** | 1 item | - | - | List of boxes to create/update. An empty array is rejected |

**Parameters — `boxes[]`:**

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `box_identifier` | string | **yes** | - | 155 (use up to **50**) | - | Your unique box identifier (e.g., `CX0236`). Validation accepts up to 155 characters, but identifiers longer than 50 characters **are not stored correctly** |
| `name` | string | **yes** | - | 155 | - | Descriptive name of the box |
| `height` | number | **yes** | 1 | - | - | Height in cm |
| `width` | number | **yes** | 1 | - | - | Width in cm |
| `depth` | number | **yes** | 1 | - | - | Depth in cm |
| `weight` | number | **yes** | - | 11 integer digits + 3 decimals | - | Weight of the empty packaging in kg. Accepted format: up to 11 integer digits and up to **3 decimal places**, `.` as separator, unsigned (e.g., `0.85`, `2.8`, `12`). Inform a value greater than zero |
| `activated` | boolean | **yes** | - | - | - | `true` for an active box, `false` for an inactive one. Also accepts `1`/`0` |
| `box_code` | **string** | no | - | 50 | - | Code of an already existing box, to update it by code. **Send it as text** (e.g., `"1"`): sent as a number, it is rejected with `422` |

> **Pay attention to the type of `box_code` in the `POST`:** here it is a **string** (`"1"`). In the quotation, in the item registry and in
> removal, it is an **integer** (`1`).

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
      "name": "Master box",
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

### Response example

**Success (`200`):**
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

**Validation error (`422`):**
```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "boxes.0.height": ["O campo boxes.0.height deve conter um número superior ou igual a 1."],
    "boxes.0.box_code": ["O campo boxes.0.box code deve ser uma string."]
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success`, `warning` (validation/rule) or `error` (unexpected) |
| `message` | string | Message translated according to the `lang` header |
| `data` | object | Only in validation errors: `field` → list of messages map. The field comes with the box index (`boxes.0.*` = first box) |

> **The `200` does not guarantee that the box was stored.** Confirm the result with `GET /v1/box` (remembering the cache of
> up to 5 minutes) or through the import report in the dashboard.

## Endpoint: remove boxes

- **Method and route:** `DELETE /v1/box`
- **When to use:** to permanently delete boxes that are no longer used.
- **Processing:** the API validates the format and responds `200`; the removal is done in the background.
  At the end, you receive **a report by e-mail** (at the account e-mail) with the total of boxes processed,
  the total removed and the list of errors.
- Removal is **permanent**. To only deactivate, use `POST /v1/box` with `activated: false`.

### Parameters

| Field | Location | Type | Required | Min. | Description |
|---|---|---|---|---|---|
| `boxes` | body | object | **yes** | - | Envelope |
| `boxes.box_ids` | body | array of integer/string | **yes** | 1 item | List of boxes to remove. **Use the `box_code` as an integer** (e.g., `1`) |

> ⚠️ **Remove by `box_code` (integer).** Although the interactive documentation accepts `box_identifier` in this list,
> removal by text identifier **is not reliable** at the moment:
> - the box may be reported as `box CX0236 not found!` even though it exists;
> - the linked items check only recognizes the link made by `box_code` — a box linked to items only
>   by `box_identifier` may be removed.
>
> Also send the `box_code` as a **number** (`1`) and not as text (`"1"`): with text, the linked items check
> may not identify the link. Query the `box_code` with `GET /v1/box` before removing.

### Minimum payload

```json
{
  "boxes": {
    "box_ids": [1]
  }
}
```

### Maximum payload

```json
{
  "boxes": {
    "box_ids": [1, 2, 3, 15]
  }
}
```

### Request example

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": [1, 2]}}'
```

### Response example

**Received (`200`):**
```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> Despite the wording, the message above only confirms **receipt**. The actual result arrives in the e-mail report.

**Unexpected error (`500`):**
```json
{
  "status": "error",
  "message": "Algo inesperado aconteceu. Tente novamente mais tarde!"
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success`, `warning` or `error` |
| `message` | string | Message translated according to the `lang` header |
| `data` | object | Only in validation errors: `field` → list of messages map |

### Removal rules (e-mail report)

Each box in `box_ids` is evaluated individually; a failure does not prevent the removal of the others.

| Rule | Message in the report when it fails |
|---|---|
| The box must exist in your account | `Box Error: box %s not found!` (`%s` = value sent) |
| The box **cannot have linked items** (item `box_code`) | `Box Error: box %s cannot be deleted, has related item` |

> To retire a box that already has linked items, change the default box of those items
> ([`PUT /v1/item`](api-itens-v1.md#endpoint-update-items)) before removing it, or just deactivate the box
> with `POST /v1/box` and `activated: false`.

## How to use the boxes in the quotation

| Form | Payload in `boxes[]` | When to use |
|---|---|---|
| By code | `{ "box_code": 1 }` | You stored the numeric code (integer) |
| By identifier | `{ "box_identifier": "CX0236" }` | You use your own packaging code |
| Manual | `{ "height": 20, "width": 20, "depth": 20, "weight": 0.3 }` | One-off packaging, not registered |
| Kit | `{ "kit": "KIT-001" }` | Box **and** items come from the kit registry |

Use **one** form per box (do not mix `box_code` with `box_identifier` or with manual dimensions in the same position).
A referenced box that does not exist generates an error in the quotation:

| Situation | Status | Message |
|---|---|---|
| `box_code`/`box_identifier` nonexistent in the platform | `422` | `O valor selecionado para o campo boxes.0.box code é inválido.` (or `box identifier`) |
| Box does not belong to your account | `400` | `A caixa :name não foi encontrada para o usuário especificado!` |

Use of registered boxes according to the grouping configured in the account (`type: items` quotation, exclusive to v2):

| Grouping | How the registered boxes are used |
|---|---|
| `default` | Each registered item must have a default box (`box_code` or `box_identifier` in the [item registration](api-itens-v1.md)). Without a linked box: `A caixa com o box_code/box_identifier :name não existe ou está desativada` (HTTP `404`) |
| `smartbox` | Uses the boxes informed with dimensions in `boxes[]` in the quotation payload. Without a valid box: `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (HTTP `400`) |
| `all_in_one` | Consolidates the items; does not depend on the default box of the item |

See the `boxes[]` section in [Quotation (v1)](api-cotacao-v1.md#parameters--boxes).

## Business rules
- Creation, update and removal are processed in the **background**; the `200` only confirms receipt.
- The listing is **cached for up to 5 minutes** per account and filter (see the pagination alert above).
- Boxes are **per account**: each user lists, uses and removes only their own.
- `box_identifier` must be unique in your account — it is what allows updating instead of duplicating. Use at most 50 characters and avoid numeric-only values.
- Registered dimensions and weight are always in **cm** and **kg**, regardless of the quotation `measurement`.
- The `active` indicator is informational in the listing: **do not reference inactive boxes** in quotations. At the moment, the quotation
  does not reliably block an inactive box referenced by `box_code`/`box_identifier`.
- A box with linked items (by the item `box_code`) is not removed.
- Removal is permanent and is reported by e-mail.

## Common errors

| Status | Meaning/Message | How to fix |
|---|---|---|
| `403` | `Has no permission!` — the profile does not have permission to list, create or remove boxes | Ask support for the permission |
| `422` | `O campo boxes é obrigatório.` — `boxes` absent or empty (`POST`/`DELETE`) | Send at least one box (`POST`) or the `boxes` envelope (`DELETE`) |
| `422` | `O campo boxes deve conter um array.` — `boxes` is not a list/object | Fix the payload structure |
| `422` | `O campo boxes.0.box identifier é obrigatório.` / `O campo boxes.0.name é obrigatório.` | Inform `box_identifier` and `name` in all boxes |
| `422` | `O campo boxes.0.box identifier não pode conter mais de 155 caracteres.` / `O campo boxes.0.name não pode conter mais de 155 caracteres.` | Shorten the value (for `box_identifier`, use up to 50) |
| `422` | `O campo boxes.0.height deve conter um número superior ou igual a 1.` (same for `width`/`depth`) | Inform dimensions ≥ 1 cm |
| `422` | `O campo boxes.0.height deve conter um valor numérico.` (same for `width`/`depth`/`weight`) | Send numbers, without a unit in the value |
| `422` | `O formato do valor informado no campo boxes.0.weight é inválido.` | Use up to 3 decimal places, `.` as separator and no sign |
| `422` | `O campo boxes.0.activated deve conter o valor verdadeiro ou falso.` | Send `true`/`false` |
| `422` | `O campo boxes.0.box code deve ser uma string.` | In the `POST`, send `box_code` as text (`"1"`) |
| `422` | `O campo boxes.0.box code não pode conter mais de 50 caracteres.` | Check the `box_code` |
| `422` | `O campo boxes.box ids é obrigatório.` / `O campo boxes.box ids deve conter um array.` (`DELETE`; in the `GET`, only the second one) | Send `box_ids` as a non-empty list |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Retry with increasing intervals; if it persists, contact support |
| Report (e-mail) | `Box Error: box %s not found!` | Check the `box_code` with `GET /v1/box` |
| Report (e-mail) | `Box Error: box %s cannot be deleted, has related item` | Unlink the box from the items or just deactivate it |

> In the validation messages, the index (`boxes.0`, `boxes.1`...) indicates the position of the box in the array, starting at `0`,
> and the `_` in the field name appears as a space (`box identifier`).

## Final notes
- Registering the boxes and linking them to the items is the prerequisite for the [v2 quotation by items](api-cotacao-v2.md#items-quotation) (`type: items`) with the `default` grouping.
- Keep the `box_identifier` values aligned with your ERP/WMS: this greatly simplifies the integration.
- Store the `box_code` returned in the listing: it is the most reliable reference for quotation, items and removal.
- To register the products that go inside the boxes, see [Items](api-itens-v1.md).
- To reference the boxes in the quotation, see [Quotation (v1)](api-cotacao-v1.md).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
