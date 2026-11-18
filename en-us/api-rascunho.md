# Draft

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
- Allows you to send an **incomplete** shipment: you send what you already have, and the platform validates it later.
- The draft is processed **asynchronously** and, when complete, evolves into quotation + shipment
  automatically.
- It is the recommended flow when the integration receives the order data in stages (e.g. the order arrives before
  the final address or the invoice).

> **Practically all fields are optional when sending.** Strict validation happens at
> dispatch time. While data is missing, the draft stays with the `incomplete` status and you can complete it.

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

## Draft vs. other flows

| Topic | Quotation + Shipment | Manifest | Draft |
|---|---|---|---|
| Payload | Complete in 2 stages | Complete in 1 call | **Partial, in as many stages as you need** |
| Validation | Synchronous, per stage | Synchronous, total | **Asynchronous, at dispatch** |
| Response | Quotation with prices | Documents ready | Receipt confirmation |
| Payment | `billed`, `gateway`, `afterward` | `billed` | **`billed` or `afterward`** |
| Later correction | New quotation | No | **Yes, via `PUT`** |

## Draft life cycle

```
POST /v1/draft
     │
     ▼
  received  ──►  validating  ──►  incomplete  ──┐
                      │                          │ PUT /v1/draft/{code}
                      ▼                          │
                   complete  ◄───────────────────┘
                      │
                      ▼
                 processing  ──►  quotation + shipment
                      │
                      └──►  error
```

| Status | Meaning | Can it be edited? |
|---|---|---|
| `received` | Draft received, waiting for validation | No |
| `validating` | Being validated | No |
| `incomplete` | **Data is missing.** The reason is in `status_descricao` | **Yes** |
| `edited` | Draft updated via `PUT`, waiting for a new validation | No |
| `complete` | All data present, ready for dispatch | No |
| `processing` | Being processed: generating the quotation and the shipment | No |
| `error` | Processing failure | No |

> **Only drafts with the `incomplete` status can be edited.** Trying to edit one in another status
> returns `400 Apenas rascunhos com o status incomplete podem ser editados`.

## Usage flow
1. `POST /v1/draft` with the data you already have.
2. Wait for the asynchronous processing.
3. `GET /v1/draft` to see the status and, if `incomplete`, read the `status_descricao`.
4. `PUT /v1/draft/{code}` to complete what was missing.
5. When it becomes `complete`, the platform generates the quotation and the shipment.
6. Follow it through the [tracking](api-rastreio.md) and download the [documents](api-documentos.md).
7. `DELETE /v1/draft` to discard drafts that will not be used.

## Endpoint: create draft

- **Method and route:** `POST /v1/draft`
- **Processing:** asynchronous.

### Parameters — payload root

No field is required, but **at least one** must be sent.

| Field | Type | Min. | Max. | Default | Values | Description |
|---|---|---|---|---|---|---|
| `object` | string | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` | What is being shipped |
| `type` | string | - | 30 | `Commercial` | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Purpose of the shipment |
| `tax` | string | - | - | account preset | `sender`, `receiver` | Who pays the taxes |
| `insurance` | boolean | - | - | account preset | - | Purchase insurance |
| `measurement` | string | - | - | `metric` | `metric`, `imperial` | Measurement system |
| `ior` | boolean | - | - | `false` | - | Importer of Record |
| `residential_delivery` | boolean | - | - | `false` | - | Residential delivery |
| `non_stackable` | boolean | - | - | `false` | - | Non-stackable packages |
| `currency_quote` | string | 3 | 3 | account preset | see [currencies](api-primeiros-passos.md#accepted-currencies) | Quotation currency |
| `currency_payment` | string | 3 | 3 | account preset | see [currencies](api-primeiros-passos.md#accepted-currencies) | Payment currency |
| `nfkey` | string | 44 | 44 | - | - | NF-e key |
| `description` | string | - | 155 | - | - | Free description of the draft |
| `channel_code` | integer | - | - | account preset | - | Code of the registered sales channel |
| `channel_identifier_number` | string | - | - | - | - | Your commercial identifier. **Recommended** |
| `carrier_code` | integer | - | - | - | - | Preferred contract, if available in the quotation |
| `address_sender` | object | - | - | account preset | - | Sender (see table) |
| `address_receiver` | object | - | - | - | - | Receiver (see table) |
| `boxes` | array | - | - | - | - | Packages (see table) |
| `items` | array | - | - | - | - | Items (see table) |
| `payment` | object | - | - | - | - | Payment (see table) |
| `biller` | object | - | - | account preset | - | Biller shown on the invoice |
| `manufacturer` | object | - | - | - | - | Manufacturer shown on the invoice |

### Parameters — `address_sender` and `address_receiver`

All optional when sending. The "Required at dispatch" columns indicate what the platform requires for the
draft to become `complete`.

| Field | Type | Min. | Max. | Required at dispatch | Description |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Code of a registered address (only in `address_sender`) |
| `country_code` | string | 2 | 2 | **Yes** | Country in ISO 3166-1 alpha-2 |
| `city` | string | - | 100 | **Yes** | City |
| `postal_code` | string | - | 12 | Yes, if `state` is not informed | Postal/ZIP code |
| `state` | string | - | - | Yes, if `state_code`/`postal_code` are not informed | State name |
| `state_code` | string | - | - | Yes, if `state`/`postal_code` are not informed | State abbreviation, validated against the country |
| `street` | string | - | 70 | **Yes** | Street |
| `name` | string | 5 | 35 | **Yes** | Contact name/legal name |
| `email` | string | - | 60 | **Yes** | Valid e-mail |
| `phone` | string | - | 25 | **Yes** | Phone |
| `type` | string | - | - | **Yes** | `pf` or `pj` (default `pj`) |
| `federal_tax_id` | string | - | 20 | Yes when the country is `BR` | CPF/CNPJ or federal document |
| `state_tax_id` | string | - | 20 | Yes when the country is `BR` | State tax registration |
| `number` | string | - | 100 | - | Building number |
| `district` | string | - | 30 | - | District |
| `complement` | string | - | 20 | - | Complement |
| `description` | string | - | 155 | - | Internal label of the address |
| `phone_extension` | string | - | 5 | - | Extension |
| `foreign` | boolean | - | - | - | Indicates whether the person is a foreigner |

### Parameters — `boxes[]`

| Field | Type | Min. | Max. | Description |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU of the registered kit. Fills in the box **and** the items of that position at dispatch |
| `box_code` | integer | - | - | Code of the registered box ([Box](api-caixa-v1.md)) |
| `box_identifier` | string | - | - | Identifier of the registered box |
| `name` | string | 4 | 155 | Box name |
| `height` | number | 1 | - | Height |
| `width` | number | 1 | - | Width |
| `depth` | number | 1 | - | Depth |
| `volume` | number | 0.001 | - | Volume (alternative to the dimensions, for `pallet`) |
| `weight` | number | 0.1 | - | Package weight. **Required when `object` is `doc`.** Up to 3 decimal places. Ignored when the box uses `kit` |
| `price` | number | 0 | - | Declared value. **Required when `object` is `doc`.** Ignored when the box uses `kit` |

### Parameters — `items[]`

| Field | Type | Min. | Max. | Required at dispatch | Description |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU of the registered item ([Items](api-itens-v1.md)) |
| `ean` | string | - | 50 | - | EAN code |
| `box_id` | integer | - | - | - | **Position** of the box in the `boxes` array (1 = first) |
| `name` | string | 6 | 35 | Yes, without `sku` | Item name |
| `description` | string | 4 | 155 | - | Item description |
| `country_code` | string | 2 | 2 | Yes, without `sku` | Country of origin of the item |
| `hscode` | string | 6 | 10 | Yes, without `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Yes, without `sku` | Unit weight |
| `quantity` | integer | 1 | 1000 | **Yes** | Quantity |
| `unit_value` | number | - | - | **Yes** | Unit value |
| `selling_unit_value` | number | - | - | Yes when `ior` is `true` | Unit selling value |
| `is_groupable` | boolean | - | - | - | Allows grouping the item with others |
| `height`, `width`, `depth` | number | 1 | - | - | Item dimensions |
| `is_foldable` | boolean | - | - | - | Foldable item (used in the `smartbox` grouping) |

> `items` becomes **required at dispatch** when `object` is `package`, `pallet` or `not_doc`.

### Parameters — `payment`

| Field | Type | Required at dispatch | Values | Description |
|---|---|---|---|---|
| `payment.type` | string | **Yes** | `billed`, `afterward` | The draft **does not accept** `gateway` |

### Minimum payload

The smallest useful payload — any field works, but the draft must have at least one:

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package"
}
```

A more realistic example, with what you normally already have at the time of the order:

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "boxes": [
    { "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Cotton t-shirt",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9
    }
  ],
  "payment": { "type": "billed" }
}
```

### Maximum payload

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "carrier_code": 1,
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Draft of the marketplace order",
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "SP distribution center",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Warehouse 2",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami",
    "description": "Miami office",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": "987654321",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Master box",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 2.5,
      "price": 150
    },
    { "box_identifier": "CX0236" },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Cotton t-shirt S",
      "description": "100% cotton t-shirt short sleeve white",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true
    }
  ],
  "biller": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "100",
    "name": "Holding Exemplo LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabrica Exemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "type": "pj"
  },
  "payment": { "type": "billed" }
}
```

### Request example

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Cotton t-shirt", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

| Status | Message | Cause |
|---|---|---|
| `400` | `Necessário informar pelo menos um parâmetro para criar o rascunho` | Empty payload |

## Endpoint: list drafts

- **Method and route:** `GET /v1/draft`
- **Items per page:** 20

| Field | Location | Type | Required | Description |
|---|---|---|---|---|
| `drafts` | body | object | no | Filter envelope |
| `drafts.drafts_ids` | body | array | no | List of draft codes. **Empty or absent returns all of them** |
| `page` | query | integer | no | Listing page |

> This `GET` accepts a JSON body for the filter. If your HTTP client does not support a body in `GET`,
> list them all and filter on your side.

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "codigo": 30,
        "codigo_usuario": 1,
        "data": {
          "channel_identifier_number": "PED-2026-0001",
          "object": "package",
          "type": "Commercial",
          "tax": "sender",
          "insurance": true,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "payment": { "type": "billed" },
          "address_receiver": {
            "country_code": "US",
            "postal_code": "33101",
            "city": "Miami",
            "street": "Main Avenue",
            "name": "John Doe",
            "email": "johndoe@example.com",
            "phone": "11992494597",
            "type": "pf"
          },
          "items": [
            {
              "name": "Cotton t-shirt",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9,
              "box_id": 1
            }
          ]
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender.country code é obrigatório. O campo address sender.street é obrigatório.",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/draft",
    "per_page": 20,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `codigo` | integer | **Draft code.** Use it in `PUT /v1/draft/{code}` and in `drafts.drafts_ids` |
| `data` | object | The draft payload, already merged with the account presets |
| `status` | string | Current status (see life cycle) |
| `status_descricao` | string | **When `incomplete`, it lists exactly what is missing.** It is the most important field to fix the draft |
| `created_at`, `updated_at` | string | Creation and last change dates |

## Endpoint: update draft

- **Method and route:** `PUT /v1/draft/{code}`
- **Restriction:** only drafts with the `incomplete` status.
- **Payload structure:** identical to the `POST`.

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `code` | path | integer | **yes** | Draft code (`codigo` in the listing) |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "address_sender": {
      "country_code": "BR",
      "postal_code": "09941070",
      "city": "Diadema",
      "street": "Rua Principal",
      "name": "Loja Exemplo LTDA",
      "email": "contato@example.com",
      "phone": "11989677465",
      "type": "pj",
      "federal_tax_id": "12345678000199",
      "state_tax_id": "123456789012"
    }
  }'
```

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso."
}
```

| Status | Message | Cause |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | The draft is in another status |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | `code` invalid or from another account |

## Endpoint: remove drafts

- **Method and route:** `DELETE /v1/draft`

| Field | Type | Required | Description |
|---|---|---|---|
| `drafts` | object | **yes** | Envelope |
| `drafts.drafts_ids` | array | **yes** | List of draft codes to remove |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> Only drafts of your own account are removed. If no code matches, the response is
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Business rules

### Validation at dispatch
When the draft is dispatched, the platform applies a **strict** set of rules. What is missing appears
in `status_descricao`. Summary of the required fields:

| Group | Required fields |
|---|---|
| General | `tax`, `insurance` |
| Sender | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`state`, `state_code` or `postal_code`) |
| Sender in BR | `federal_tax_id` and `state_tax_id` |
| Receiver | The same fields as the sender |
| Items (`object` ≠ `doc`) | `items` with `quantity` and `unit_value`; and `name`, `country_code`, `hscode`, `weight` when there is no `sku` |
| Boxes (`object` = `doc`) | `boxes[].weight` and `boxes[].price` |
| Payment | `payment.type` (`billed` or `afterward`) |

### Manufacturer
If you send `manufacturer` **without** `address_code`, the fields `name`, `country_code`, `state_code`,
`postal_code`, `city`, `street`, `email`, `phone` and `type` become required at dispatch.

For items with an `hscode` starting with **`64`** (footwear), the `manufacturer.name` is required.

### Account presets
The presets are merged into the draft. **The payload always beats the preset**, with these exceptions:
- Informing `address_sender.country_code` discards the sender address preset.
- If the account has the tax locked (`tax locked`), the value is forced to `sender`.
- The biller preset is only applied when you send an empty `biller`.

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Draft empty, not found or not editable | Send at least one field, check the `code` and the status |
| `403` | No permission for the action | Ask support for the permission |
| `422` | Field validation | Fix the fields listed in `data` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- The **`status_descricao` field is your guide**: it tells exactly what is missing for the draft to move forward.
- Always inform `channel_identifier_number`: it is what ties the draft to your order and allows
  querying the shipment later.
- The draft **does not accept** payment via `gateway`. For that scenario use
  [quotation](api-cotacao-v1.md) + [shipment](api-remessa-v1.md).
- For the opposite flow (everything validated at once, immediate documents), see [Manifest](api-manifesto.md).
