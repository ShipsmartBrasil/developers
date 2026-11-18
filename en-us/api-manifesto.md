# Manifest

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
- Runs **quotation + shipment + document generation in a single synchronous call**.
- The response already brings the tracking number and the documents (AWB, and the others when applicable).
- It is the recommended flow for operations that need the label **immediately**, without waiting for
  asynchronous processing.

> ⚠️ **The manifest is synchronous and irreversible.** Since it quotes and dispatches in the same request, there is no
> intermediate review stage. Validate the payload thoroughly before sending it.
>
> ⚠️ **It only accepts billed payment** (`payment.type: billed`).

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

## Manifest vs. quotation + shipment

| Topic | Quotation + Shipment | Manifest |
|---|---|---|
| Calls | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Carrier choice | You choose from the returned list | The platform resolves it (or you fix it with `carrier_code`/`slug`) |
| Documents | Later query at `/docs` | **They come in the response** |
| Payment | `billed`, `gateway`, `afterward` | **Only `billed`** |
| Response type | Asynchronous | **Synchronous** |
| Review before dispatch | Yes | No |
| Required fields | Fewer (split into 2 stages) | **More** (everything at once) |

Use the manifest when: the operation is high volume, the carrier is already defined by a business
rule and you need to print the label on the spot.

## Usage flow
1. Assemble the complete payload (addresses, boxes, items, payment).
2. `POST /v1/manifest?format=pdf` (or `zpl`).
3. Print the label received in `data.docs`.
4. Use the `data.shipping` for [tracking](api-rastreio.md), [documents](api-documentos.md) and
   [pickup scheduling](api-agendamento.md).

## Endpoint: create manifest

- **Method and route:** `POST /v1/manifest`

### Query parameters

| Name | Location | Type | Required | Default | Values | Description |
|---|---|---|---|---|---|---|
| `format` | query | string | no | all | `pdf`, `zpl` | Format of the documents in the response. Empty returns all available formats |

> `format` can also be sent **in the body** of the payload. When sent in both places,
> **the body value prevails**.

### Parameters — payload root

| Field | Type | Required | Min. | Max. | Values / Default | Description |
|---|---|---|---|---|---|---|
| `object` | string | **yes** | - | - | `doc`, `package`, `pallet`, `not_doc` | What is being shipped |
| `type` | string | **yes** | - | 30 | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Purpose of the shipment |
| `tax` | string | **yes** | - | - | `sender`, `receiver` | Who pays the taxes |
| `insurance` | boolean | **yes** | - | - | - | Purchase insurance |
| `residential_delivery` | boolean | **yes** | - | - | - | Delivery to a residential address |
| `non_stackable` | boolean | **yes** | - | - | - | Non-stackable packages |
| `measurement` | string | **yes** | - | - | `metric`, `imperial` | Measurement system |
| `ior` | boolean | **yes** | - | - | - | Importer of Record |
| `currency_quote` | string | **yes** | 3 | 3 | see [currencies](api-primeiros-passos.md#accepted-currencies) | Quotation currency |
| `currency_payment` | string | **yes** | 3 | 3 | see [currencies](api-primeiros-passos.md#accepted-currencies) | Payment currency |
| `address_sender` | object | **yes** | - | - | - | Sender (see table) |
| `address_receiver` | object | **yes** | - | - | - | Receiver (see table) |
| `boxes` | array | **yes** | 1 item | - | - | Packages (see table) |
| `payment` | object | **yes** | - | - | - | Payment (see table) |
| `items` | array | conditional | - | - | - | Required when `object` is **not** `doc` and no box uses `kit`. **Forbidden when `object` is `doc`** |
| `carrier_code` | integer | no | - | - | - | Fixes the contract to be used |
| `slug` | string | no | - | 20 | - | Fixes the contract by its text identifier |
| `channel_code` | integer | no | - | - | - | Code of the registered sales channel |
| `channel_identifier_number` | string | no | - | - | - | Your commercial identifier. **Recommended** |
| `nfkey` | string | no | 44 | 44 | - | NF-e key |
| `description` | string | no | 5 | 155 | - | Free description of the manifest |
| `declared_freight` | object | no | - | - | - | Declared freight (see table) |
| `biller` | object | no | - | - | - | Biller shown on the invoice |
| `manufacturer` | object | no | - | - | - | Manufacturer shown on the invoice |
| `extra` | object | no | - | - | - | Auxiliary data in key/value pairs (see table) |
| `format` | string | no | - | - | `pdf`, `zpl` | Alternative to the query string |
| `size` | string | no | - | - | `label`, `letter` | Label size |

### Parameters — `address_sender` and `address_receiver`

Unlike the v1/v2 shipment, here the address is informed in **full**, because the quotation happens
in the same call.

**Form A — address already registered:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this form | Code of the registered address. **Do not send any other address field along with it** |

**Form B — manual address:**

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `country_code` | string | **yes** | 2 | 2 | - | Country in ISO 3166-1 alpha-2 |
| `city` | string | conditional | 1 | 100 | - | City. Required if `postal_code` is not informed |
| `postal_code` | string | conditional | - | 12 | - | Postal/ZIP code. Required if `state` and `state_code` are not informed |
| `state` | string | conditional | - | 100 | - | State name. Required if `state_code` and `postal_code` are not informed |
| `state_code` | string | conditional | 1 | 10 | - | State abbreviation, validated against the country. Required if `state` and `postal_code` are not informed |
| `street` | string | **yes** | 5 | 70 | - | Street |
| `name` | string | **yes** | 1 | 70 | - | Contact name/legal name |
| `email` | string | **yes** | - | 60 | - | Valid e-mail |
| `type` | string | **yes** | - | - | `pj` | `pf` or `pj` |
| `foreign` | boolean | **yes** | - | - | - | Indicates whether the person is a foreigner (used to validate the `federal_tax_id`) |
| `federal_tax_id` | string | conditional | - | 20 | - | **Required when the country is `BR` or `AR`** |
| `phone` | string | no | 1 | 25 | - | Phone |
| `phone_extension` | string | no | - | 5 | - | Extension |
| `number` | string | no | 1 | 6 | - | Building number |
| `district` | string | no | 1 | 30 | - | District |
| `complement` | string | no | 1 | 20 | - | Complement |
| `description` | string | no | 1 | 155 | - | Internal label of the address |
| `state_tax_id` | string | no | - | 20 | - | State tax registration |
| `company_name` | string | no | 5 | 50 | - | **Only in `address_sender`.** Legal name required by some FedEx shipments |

### Parameters — `boxes[]`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `kit` | string | alternative to the other forms | - | 50 | SKU of the registered kit. Fills in the box **and** the items of that position |
| `box_code` | integer | alternative | - | - | Code of the registered box ([Box](api-caixa-v1.md)) |
| `box_identifier` | string | alternative | - | 50 | Identifier of the registered box |
| `name` | string | **yes** without `kit`/`box_code`/`box_identifier` | 5 | 155 | Box name |
| `height` | number | **yes** without `kit`/`box_code`/`box_identifier` | 1 | - | Height (cm in `metric`, in in `imperial`) |
| `width` | number | **yes** without `kit`/`box_code`/`box_identifier` | 1 | - | Width |
| `depth` | number | **yes** without `kit`/`box_code`/`box_identifier` | 1 | - | Depth |
| `weight` | number | **yes** when `object` is `doc` | 0.001 | - | Package weight (kg in `metric`, lb in `imperial`) |
| `price` | number | **yes** when `object` is `doc` | 0 | - | Declared value of the package |

### Parameters — `items[]`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `box_id` | integer | **yes** | - | - | **Position** of the box in the `boxes` array (1 = first) |
| `quantity` | integer | **yes** | 1 | 99 | Item quantity |
| `sku` | string | alternative to the manual fields | 1 | 50 | SKU of the registered item ([Items](api-itens-v1.md)) |
| `name` | string | **yes** without `sku` | 6 | 50 | Item name |
| `country_code` | string | **yes** without `sku` | 2 | 2 | Country of origin of the item |
| `hscode` | string | **yes** without `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **yes** without `sku` | 0.001 | 99 | Unit weight |
| `unit_value` | number | **yes** without `sku` | 0 | - | Unit value |
| `description` | string | no | 5 | 450 | Detailed description of the item |
| `selling_unit_value` | number | conditional | 0 | - | Unit selling value. **Necessary when `ior` is `true`** |

> **Mind the differences compared to the quotation:** here `quantity` has a maximum of **99** (not 1000) and
> `description` accepts up to **450** characters (not 155).

### Parameters — `payment`

| Field | Type | Required | Values | Description |
|---|---|---|---|---|
| `payment` | object | **yes** | - | Envelope |
| `payment.type` | string | **yes** | **`billed`** | The manifest accepts **only** billed payment |

### Parameters — `declared_freight`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `declared` | boolean | no | - | - | Enables the declared freight. With `true`, the value can be `0` and the taxes are calculated with zero declared freight |
| `value` | number | no | 0 | - | Declared freight value |
| `currency` | string | **yes** when `value` is informed | 3 | 3 | Currency of the declared freight |

### Parameters — `extra`

Auxiliary key/value pairs, passed on to the manifest process.

| Rule | Detail |
|---|---|
| Structure | **Flat** object (one level, no nesting) |
| Keys | Only lowercase letters and `_` |
| Values | Text, integer or decimal |

| Known key | Type | Rule | Description |
|---|---|---|---|
| `federal_tax_provision` | number | - | Federal tax provision |
| `state_tax_provision` | number | - | State tax provision |
| `usd_rate_provision` | number | > 0 | US dollar exchange rate provision |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Parameters — `biller` and `manufacturer`

- `biller`: same format described in [Shipment (v1)](api-remessa-v1.md#parameters--biller-invoicing-party).
  In the manifest, `biller.name` accepts up to **355** characters.
- `manufacturer`: same format described in [Quotation (v1)](api-cotacao-v1.md#parameters--manufacturer).

### Minimum payload

Document manifest (`object: doc`) — `items` is forbidden in this case:

```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Documents envelope",
      "height": 2,
      "width": 23,
      "depth": 32,
      "weight": 0.3,
      "price": 50
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Maximum payload

```json
{
  "format": "pdf",
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "slug": "fedex-br",
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
  "description": "Manifest of the march batch",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
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
    "company_name": "Loja Exemplo Comercio LTDA",
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
    "foreign": true
  },
  "boxes": [
    {
      "name": "Master box 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "Cotton t-shirt S",
      "description": "100% cotton t-shirt, short sleeve, white, size S",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Sport sneaker",
      "description": "Men's running sneaker size 44",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9
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
  "payment": {
    "type": "billed"
  },
  "extra": {
    "federal_tax_provision": 10,
    "state_tax_provision": 7,
    "usd_rate_provision": 5.42
  }
}
```

### Request example

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "residential_delivery": false,
    "non_stackable": false,
    "measurement": "metric",
    "ior": false,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "type": "pj", "federal_tax_id": "12345678000199", "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "type": "pf", "foreign": true},
    "boxes": [{"name": "Documents envelope", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "271249639538",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `data.shipping` | string (UUID) | **Shipment identifier.** Use it in tracking, documents and scheduling |
| `data.carrier` | string | Carrier that processed the shipment |
| `data.tracking_id` | string | Tracking number (AWB) |
| `data.docs` | object | Generated documents, in the format `name → { url, base64 }` |

> Additional documents (invoice, packing list, declaration) may be generated in the background.
> Check [Shipment Documents](api-documentos.md) a few moments later to get the complete list.

## Carrier error

When the carrier refuses the shipment, the response is `400` and brings its raw return:

```json
{
  "status": "error",
  "message": "[1234] CARRIER - Carrier error message",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Field | Description |
|---|---|
| `message` | `[code] CARRIER - message` returned by the carrier |
| `data` | Raw payload from the carrier. **The structure varies per carrier** |

> Log `message` and `data` in your logs: it is the only source of the technical reason for the refusal.

## Business rules
- **Payment:** `billed` only. Any other value is rejected in the validation.
- **`items` and `object: doc`:** with `object: doc`, the `items` field **cannot** be sent.
  With any other `object`, `items` is required — except when some box uses `kit`.
- **Kits:** when a box informs `kit`, the box and the items of that position come from the registration,
  and `boxes[].weight`/`boxes[].price` are ignored.
- **`state_code`:** validated against the `country_code` of the same address.
- **`federal_tax_id`:** required for `BR` and `AR`.
- **`address_code`:** when used, **no** other address field can be sent in the same object.
- **Account presets** are applied in the same way as in the draft: the payload always beats the preset.
  Informing `declared_freight` discards the declared freight preset; informing
  `address_sender.country_code` discards the sender address preset.
- **`not_doc`** is accepted for compatibility and internally converted to `package`.

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Carrier error or business rule | Read `message` and `data`; adjust the payload or contact support |
| `403` | No permission to create a manifest | Ask support for the permission |
| `422` | Field validation | Fix the fields listed in `data` |
| `500` | Unexpected error | **Before retrying**, check in [Tracking](api-rastreio.md) whether the shipment was created |

> ⚠️ **Never repeat a manifest automatically after a `500`.** Since the call actually creates the shipment,
> a repetition may generate a duplicate shipment. Check first by the `channel_identifier_number`.

## Final notes
- Always inform `channel_identifier_number`: it is your protection against duplication and the way to locate
  the shipment if the response gets lost.
- The manifest concentrates a lot of responsibility in one call. For integrations that can wait,
  the [quotation](api-cotacao-v1.md) + [shipment](api-remessa-v1.md) flow is safer.
- For the opposite flow (partial payload, validated later), see [Draft](api-rascunho.md).
