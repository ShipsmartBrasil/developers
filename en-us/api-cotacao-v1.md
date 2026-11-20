# Quotation (v1)

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Calculates freight, insurance and taxes for an international shipment, across all carriers enabled for your account.
- It is the **mandatory first step** of the flow: the shipment, tracking, documents and pickup scheduling
  are all tied to the quotation code (`quotation`).
- v1 is the complete version: it receives the address (country, city, ZIP code/state) already in the quotation, validates that address
  and has an endpoint to re-query the result.
- It covers five cargo scenarios: document, simple package, advanced package, pallet by LWH and pallet by m³ —
  see [Quotation scenarios](#quotation-scenarios). The quotation **by items** (only `sku`, no boxes) is exclusive to
  [Quotation (v2)](api-cotacao-v2.md#items-quotation).

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
Details in [Getting Started](api-primeiros-passos.md).

## Usage flow
1. Choose the [quotation scenario](#quotation-scenarios) that matches your cargo (`object` + `type`).
2. `POST /v1/quotation` with origin, destination, object, type and boxes/items.
3. The response already brings the calculated result (`data.carriers`) and the `data.quotation` code.
4. If you need to re-query it (or change the level of detail), use `GET /v1/quotation/{quotation}/{level}`.
5. Check whether the quotation is [eligible for shipment](#quotations-ineligible-for-shipment).
6. Choose a carrier from the list and use `code` (or `slug`) + `quotation` to create the
   [shipment](api-remessa-v1.md) **before** the `valid_until`.

```
POST /v1/quotation  ──►  quotation + carriers[]
                              │
                              ▼
                   POST /v1/shipping  (carrier_code | slug)
```

## Key concepts

### `object` — what is being shipped
| Value | Meaning | Notes |
|---|---|---|
| `doc` | Document | Quoted by volume, providing the `weight` and `price` of each envelope/box |
| `package` | Package/box | Most common flow |
| `pallet` | Pallet | Accepts dimensions (LWH) **or** `volume` in m³/ft³ |
| `not_doc` | Legacy | Kept for compatibility; it is converted internally to `package`. **Do not use it in new integrations** |

### `type` — how the calculation is built
| Value | What you send | When to use |
|---|---|---|
| `simple` | Only `boxes[]` with `weight` and `price` | Documents and quick freight simulations. **Not eligible for shipment when `object` is `package` or `pallet`** |
| `advanced` | `boxes[]` (dimensions) + `items[]` with `box_id` pointing to the box position | Standard flow for shipment: you control how the items are distributed across the boxes |
| `items` | **Not available in v1** | Exclusive to [Quotation (v2)](api-cotacao-v2.md#items-quotation) |

### Eligibility matrix

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Quotation and shipment | Not recommended | ❌ v2 only |
| `package` | ⚠️ Simulation only (ineligible for shipment) | ✅ Quotation and shipment | ❌ v2 only |
| `pallet` | ⚠️ Simulation only (ineligible for shipment) | ✅ Quotation and shipment | ❌ v2 only |

> **Attention:** `type: simple` with `object: package` or `object: pallet` is for simulation only.
> When trying to create the shipment, the API responds `403` with `Cotação Simples é inelegível para remessa!`.
> Details in [Quotations ineligible for shipment](#quotations-ineligible-for-shipment).

### `tax` — who pays the taxes
| Value | Meaning |
|---|---|
| `sender` | DDP: the sender pays taxes and fees at destination |
| `receiver` | DDU: the receiver pays taxes and fees at destination |

> If your account has a **tax preset** configured, it **replaces** the value sent in `tax`.
> See [Presets applied to the quotation](#presets-applied-to-the-quotation).

### `measurement` — measurement system
| Value | Dimensions | Weight | Volume |
|---|---|---|---|
| `metric` (default) | centimeters | kilograms | m³ |
| `imperial` | inches | pounds | ft³ |

### `level` — level of detail of the result
| Value | Return |
|---|---|
| `simple` (default) | Lean list of carriers with totals |
| `detailed` | Includes the exchange rate and the detailed breakdown of freight, insurance and tax, with the subsidy rules applied |

Sent as a **query string** in the `POST` and as a **path segment** in the `GET`.

## Endpoint: create quotation

- **Method and route:** `POST /v1/quotation`
- **Optional query string:** `level=simple|detailed`
- **When to use:** whenever you need price, transit time and taxes, and before creating any shipment.

### Parameters — payload root

| Field | Type | Required | Min. | Max. | Default | Accepted values | Description |
|---|---|---|---|---|---|---|---|
| `object` | string | **yes** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | What is being shipped |
| `type` | string | **yes** | - | - | - | `simple`, `advanced` | How the calculation is built (`items` is exclusive to v2) |
| `tax` | string | **yes** | - | - | - | `sender`, `receiver` | Who pays the taxes (may be replaced by the account preset) |
| `insurance` | boolean | **yes** | - | - | - | `true`, `false` | Purchase insurance (may be enabled automatically by the account) |
| `currency_quote` | string | **yes** | 3 | 3 | - | see [currencies](api-primeiros-passos.md#accepted-currencies) | Currency in which the quotation is presented and in which `unit_value`/`price` are provided |
| `currency_payment` | string | **yes** | 3 | 3 | - | see [currencies](api-primeiros-passos.md#accepted-currencies) | Currency in which the payment will be made |
| `address_sender` | object | **yes** | - | - | - | - | Origin (see table below) |
| `address_receiver` | object | **yes** | - | - | - | - | Destination (see table below) |
| `boxes` | array | **yes** | - | - | - | - | Packages of the quotation (see table below) |
| `items` | array | conditional | - | - | - | - | Required when `type` is **not** `simple` and no box uses `kit` |
| `measurement` | string | no | - | - | account preset or `metric` | `metric`, `imperial` | Measurement system for dimensions, weights and volumes |
| `ior` | boolean | no | - | - | `false` | `true`, `false` | Importer of Record. When `true`, `items[].selling_unit_value` starts being considered |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Delivery to a residential address |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Non-stackable packages |
| `description` | string | no | - | 155 | `null` | - | Free description of the quotation, for your reference |
| `channel_code` | integer | no | - | - | account preset | - | Code of the sales channel registered and enabled for your user |
| `channel_identifier_number` | string | no | - | - | `null` | - | Your commercial identifier (order no.). **Recommended**: allows you to query status/documents and cancel by channel |
| `channel_order_number` | string | no | - | 100 | `null` | - | Order number in the sales channel |
| `declared_freight` | object | no | - | - | account preset | - | Declared freight (see table below). Requires a specific permission |
| `manufacturer` | object | no | - | - | `null` | - | Manufacturer shown on the Commercial Invoice (see table below) |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Level of detail of the result |

### Parameters — `address_sender` and `address_receiver`

There are **two ways** to provide the address. Choose one and do not mix them:

**Option A — already registered address:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this option | Code of an address registered in your account panel. When used, do not send the other fields |

**Option B — manual address:**

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `country_code` | string | **yes** | 2 | 2 | Country in ISO 3166-1 alpha-2 (e.g. `BR`, `US`) |
| `city` | string | **yes** | - | 100 | City |
| `postal_code` | string | conditional | - | 12 | ZIP/postal code. Required if `state` is not provided. **Always required for addresses in Brazil** |
| `state` | string | conditional | - | - | State/province name. Required if `state_code` and `postal_code` are not provided |
| `state_code` | string | conditional | - | - | State/province abbreviation (e.g. `SP`, `FL`). Required if `state` and `postal_code` are not provided |

> **Rule of thumb:** always provide `country_code` + `city` + `postal_code` (+ `state_code`, when you know it).
> This combination works for every country and is the one that produces the most accurate calculation.

> **Registered sender required for some account types:** individual accounts and business accounts
> with a single distribution center must provide the sender using **Option A** (`address_code`).
> Without it, the API responds `406` with `Codigo de endereço do remetente não encontrado!`.

### Parameters — `boxes[]`

Each element of the array represents a package (box, envelope or pallet). There are **three ways** to describe the package:

**Option A — registered kit** (fills in the box **and** the items automatically):

| Field | Type | Required | Max. | Description |
|---|---|---|---|---|
| `kit` | string | **yes** in this option | 50 | SKU of the registered kit. The box and the items of that position are filled in from the kit registration |

**Option B — registered box:**

| Field | Type | Required | Description |
|---|---|---|---|
| `box_code` | integer | **yes** in this option | Code of the registered box ([Box](api-caixa-v1.md)) |
| `box_identifier` | string | alternative to `box_code` | Identifier of the registered box (e.g. `CX0236`) |

**Option C — manually provided box:**

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `name` | string | no | 4 | 60 | Name/label of the package (used in error messages) |
| `height` | number | **yes** (except pallet with `volume`) | 1 | - | Height |
| `width` | number | **yes** (except pallet with `volume`) | 1 | - | Width |
| `depth` | number | **yes** (except pallet with `volume`) | 1 | - | Depth (length) |
| `volume` | number | conditional | 0.001 | - | **Only when `object` is `pallet`.** Alternative to height/width/depth, in m³ (`metric`) or ft³ (`imperial`) |

**Fields valid in options B and C:**

| Field | Type | Required | Min. | Description |
|---|---|---|---|---|
| `weight` | number | **yes** when `object` is `doc`/`package` **and** `type` is `simple` | > 0 | Weight of the package. In `advanced`, when omitted, it is the sum of the weights of the items in that box |
| `price` | number | **yes** when `object` is `doc`/`package` **and** `type` is `simple` | > 0 | Declared value of the package. In `advanced`, when omitted, it is the sum of `unit_value × quantity` of the items in that box |

> `weight` and `price` are **ignored** when the box uses `kit`.

### Parameters — `items[]`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `quantity` | integer | **yes** | 1 | 1000 | Quantity of the item |
| `unit_value` | number | **yes** | > 0 | - | Unit value in the quotation currency. Zero or negative is rejected |
| `box_id` | integer | **yes** | 1 | no. of boxes | **Position** of the box in the `boxes` array (1 = first box, 2 = second...). It is not the `box_code` |
| `sku` | string | no | - | 50 | SKU of the registered item ([Items](api-itens-v1.md)). Omitted fields are completed from the registration |
| `ean` | string | no | - | 50 | EAN code of the item |
| `nsmp` | string | no | - | 50 | *Non Standard Manufacturer Product Id*: the manufacturer's product identifier. Printed on the invoice and used in the regulatory information required by some carriers for destinations in the European Union |
| `name` | string | **yes** when `sku` is not provided | 6 | 50 | Item name (shown on the invoice) |
| `description` | string | no | - | 155 | Detailed description of the item |
| `country_code` | string | **yes** when `sku` is not provided | 2 | 2 | Country of origin/manufacture of the item |
| `hscode` | string | **yes** when `sku` is not provided | 6 | 10 | NCM/HS Code of the item. Defines the tax table applied |
| `weight` | number | **yes** when `sku` is not provided | 0.001 | - | Unit weight of the item |
| `selling_unit_value` | number | no | - | - | Unit selling value. Considered when `ior` is `true`; if omitted, it uses the value from the item registration and, if there is none, the `unit_value` |
| `is_groupable` | boolean | no | - | - | Default `true`. Used in the automatic box groupings of the quotation by items (v2) |

> **Items with the same `name`** generate a `warning` in the response
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> The quotation is made, but it is worth differentiating the names to avoid being held at customs.

### Parameters — `declared_freight`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Declared freight value |
| `currency` | string | **yes** when `value` is provided | 3 | 3 | Currency of the declared freight. Same list as `currency_quote` |

> Declared freight is only applied if your profile has the **declared freight permission**. Without it, the field is
> silently ignored (the quotation is made normally, without declared freight).

### Parameters — `manufacturer`

Manufacturer data printed on the Commercial Invoice. Send `address_code` **or** the manual fields.

| Field | Type | Recommended | Min. | Max. | Description |
|---|---|---|---|---|---|
| `address_code` | integer | alternative to the manual fields | - | - | Registered address code. When used, the registration data replaces the manual fields |
| `name` | string | **yes** without `address_code` | 5 | 30 | Company name/manufacturer name |
| `country_code` | string | **yes** without `address_code` | 2 | 2 | Country |
| `state_code` | string | **yes** without `address_code` | 1 | 10 | State abbreviation |
| `state` | string | no | - | 155 | State name |
| `postal_code` | string | **yes** without `address_code` | - | 12 | ZIP/postal code |
| `city` | string | **yes** without `address_code` | - | 100 | City |
| `street` | string | **yes** without `address_code` | - | 70 | Street |
| `number` | string | no | - | 6 | Number |
| `district` | string | no | - | 30 | District |
| `complement` | string | no | - | 20 | Complement |
| `email` | string | **yes** without `address_code` | - | 60 | Email |
| `phone` | string | **yes** without `address_code` | - | 25 | Phone |
| `phone_extension` | string | no | - | 6 | Extension |
| `type` | string | **yes** without `address_code` | - | - | `pf` or `pj` |
| `federal_tax_id` | string | no | - | 20 | CPF/CNPJ or equivalent |
| `state_tax_id` | string | no | - | 20 | State registration |

> The API **does not reject** an incomplete `manufacturer` in the quotation, but the data is printed on the invoice:
> send all the fields marked as recommended to avoid problems in customs clearance.
> For items with an `hscode` starting with `64` (footwear), the manufacturer name is required by customs.

### Minimum payload

The smallest payload accepted by the endpoint (document quotation). The minimum payloads for each cargo type are
in [Quotation scenarios](#quotation-scenarios).

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "postal_code": "33101"
  },
  "boxes": [
    {
      "height": 30,
      "width": 25,
      "depth": 2,
      "weight": 0.5,
      "price": 10
    }
  ]
}
```

### Maximum payload

All the fields accepted by the endpoint, with the three box options:

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "channel_order_number": "334455",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": false,
  "ior": true,
  "description": "Marketplace order, october batch",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
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
    {
      "box_identifier": "CX0236"
    },
    {
      "box_code": 3
    },
    {
      "kit": "KIT-001"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "Cotton t-shirt S",
      "description": "100% cotton t-shirt, short sleeve, white",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true
    },
    {
      "box_id": 2,
      "name": "Sport sneaker",
      "description": "Mens running sneaker",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9,
      "is_groupable": false
    },
    {
      "box_id": 3,
      "sku": "CNC-010-M",
      "quantity": 2,
      "unit_value": 15
    }
  ],
  "manufacturer": {
    "name": "Fabrica Exemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Warehouse 2",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  }
}
```

> In `type: advanced`, **every box needs at least one item** (except the `kit` box, which already brings its own).
> That is why the example associates an item with boxes 1, 2 and 3.

### Request example

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/quotation?level=detailed' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "USD",
    "address_sender": {"country_code": "BR", "city": "Diadema", "postal_code": "09941070"},
    "address_receiver": {"country_code": "US", "city": "Miami", "state_code": "FL", "postal_code": "33101"},
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Cotton t-shirt", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}]
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Cotação Realizada com sucesso!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2026-02-21 14:28:10",
    "description": null,
    "channel": "Marketplace Exemplo",
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_quote_exchange_value": "1",
    "currency_payment": "EUR",
    "currency_payment_exchange_value": "0.92",
    "warnings": [
      "Itens com descrições iguais podem causar problemas na fiscalização aduaneira"
    ],
    "carriers": [
      {
        "code": 1,
        "slug": "dhl-br",
        "name": "DHL",
        "url_image": "https://files.example.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
        "freight": "181.95",
        "freight_final": "181.95",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.45",
        "insurance_final": "11.45",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "28153.894",
        "tax_final": "28153.894",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      }
    ]
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `data.quotation` | string (UUID) | **Quotation code.** Store it: it is the key to the whole flow |
| `data.finished` | boolean | `true` when the quotation **has already been used** to create a shipment; `false` while it is still available |
| `data.date` | string | Date/time of the quotation (`YYYY-MM-DD HH:MM:SS`) |
| `data.description` | string | Description sent in `description` |
| `data.channel` | string | Name of the sales channel, when there is a `channel_code` |
| `data.channel_identifier_number` | string | Commercial identifier sent in the quotation |
| `data.object` / `data.type` / `data.tax` / `data.insurance` | - | Values actually used in the calculation (after account presets) |
| `data.currency_quote` / `data.currency_payment` | string | Quotation and payment currencies |
| `data.currency_quote_exchange_value` | string | **Only in `detailed`.** Exchange rate applied to the quotation currency |
| `data.currency_payment_exchange_value` | string | **Only in `detailed`.** Exchange rate applied to the payment currency |
| `data.warnings` | array | **Optional.** Only appears when there are alerts (see [Quotation alerts](#quotation-alerts-warnings)) |
| `data.carriers[]` | array | Calculated shipping options |
| `carriers[].code` | integer | Contract code. Use it in `carrier_code` in the shipment |
| `carriers[].slug` | string | Text identifier of the contract. Alternative to `carrier_code` in the shipment |
| `carriers[].name` | string | Commercial name of the carrier/contract |
| `carriers[].url_image` | string | Carrier logo |
| `carriers[].valid_until` | string | **Quotation validity.** The shipment must be created before this date/time |
| `carriers[].transit_days` | integer | Estimated transit time in days |
| `carriers[].weight_details.type` | string | `real` (actual weight) or `cubed` (dimensional weight) — which one was used for pricing |
| `carriers[].weight_details.weight` | string | Weight considered |
| `carriers[].freight` / `freight_final` | string | Freight before / after administrative fees |
| `carriers[].insurance` / `insurance_final` | string | Insurance before / after administrative fees |
| `carriers[].tax` / `tax_final` | string | Taxes before / after administrative fees |
| `carriers[].*_subsidy_rule_code` | integer | **Only in `detailed`.** Code of the subsidy rule applied to freight, insurance or tax (`null` when there is none) |
| `carriers[].*_details` | object | **Only in `detailed`.** Detailed breakdown of the freight, insurance or tax (`null` when there is none) |
| `carriers[].currency_quote_price` | string | Total in the quotation currency, before administrative fees |
| `carriers[].currency_quote_amount` | string | Total to pay in the quotation currency |
| `carriers[].currency_payment_amount` | string | Total to pay in the payment currency |

## Endpoint: retrieve the quotation result

- **Method and route:** `GET /v1/quotation/{quotation}/{level}`
- **When to use:** to re-query an already calculated result, change the level of detail or find out whether the quotation
  has already been used (`finished`).

### Parameters

| Parameter | Location | Type | Required | Values | Description |
|---|---|---|---|---|---|
| `quotation` | path | string | **yes** | UUID | Code returned on creation |
| `level` | path | string | no | `simple`, `detailed` | Default `simple` |

### Request example

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

### Response example

```json
{
  "status": "success",
  "message": "Resultado da cotação solicitada!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "object": "package",
    "type": "advanced",
    "carriers": []
  }
}
```

> The response has the same `data` structure as the `POST` (except `warnings`).

| Status | Message | Cause |
|---|---|---|
| `400` | `Não foi encontrada a cotação especificada!` | The `quotation` does not exist or does not belong to your account |
| `400` | `Nivel de detalhe da cotação não existe` | `level` other than `simple`/`detailed` |
| `403` | `Has no permission!` | Your profile does not have permission to query quotations |

## Quotation scenarios

Each scenario below shows what changes in the payload, the minimum payload and the maximum payload **for that scenario**.
The common root fields (currencies, tax, insurance, channel, declared freight, etc.) follow the
[payload root table](#parameters--payload-root).

| Scenario | `object` | `type` | `boxes[]` | `items[]` | Eligible for shipment |
|---|---|---|---|---|---|
| [Document](#document-quotation) | `doc` | `simple` | Dimensions + `weight` + `price` | Not used | ✅ |
| [Package — simple](#package-box-quotation--simple) | `package` | `simple` | Dimensions + `weight` + `price` | Not used | ❌ simulation only |
| [Package — advanced](#package-box-quotation--advanced) | `package` | `advanced` | Dimensions (optional weight/value) | Required, with `box_id` | ✅ |
| [Pallet — by LWH](#pallet-quotation--by-lwh) | `pallet` | `advanced` | `height` + `width` + `depth` | Required, with `box_id` | ✅ |
| [Pallet — by m³](#pallet-quotation--by-m³) | `pallet` | `advanced` | `volume` | Required, with `box_id` | ✅ |
| [Items](#items-quotation) | `package` | `items` | - | - | **Exclusive to v2** |

### Document Quotation

- **When to use:** shipping papers/documents with no relevant commercial value (contracts, certificates, printed matter).
- **How to build it:** `object: doc` + `type: simple`. Each package (envelope/box) provides dimensions, `weight` and `price`.
- **Does not use** `items[]`, `ior` or `manufacturer`: documents do not go through the per-item tax calculation.
- **It is eligible for shipment.**

#### Minimum payload

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
  "boxes": [
    { "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 }
  ]
}
```

#### Maximum payload

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0002",
  "channel_order_number": "334456",
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Shipping of signed contracts",
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "declared_freight": { "value": 20, "currency": "BRL" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
  "boxes": [
    { "name": "A4 envelope", "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 },
    { "box_code": 7, "weight": 1.2, "price": 15 }
  ]
}
```

> In `doc`, `weight` and `price` are required **in every package**, even when the box is registered (`box_code`/`box_identifier`).

### Package (Box) Quotation — simple

- **When to use:** quick freight simulation when you only know the weight and value of each box.
- **How to build it:** `object: package` + `type: simple`. Each box provides dimensions, `weight` and `price`. No items.
- ⚠️ **It is not eligible for shipment.** To actually ship, redo the quotation as
  [Package — advanced](#package-box-quotation--advanced) (or, in v2, [Items](api-cotacao-v2.md#items-quotation)).

#### Minimum payload

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
  "boxes": [
    { "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 }
  ]
}
```

#### Maximum payload

```json
{
  "channel_code": 1,
  "channel_identifier_number": "SIM-2026-0001",
  "channel_order_number": "334457",
  "object": "package",
  "type": "simple",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": true,
  "description": "Freight simulation for the storefront",
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "declared_freight": { "value": 30, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
  "boxes": [
    { "name": "Medium box", "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 },
    { "box_identifier": "CX0236", "weight": 3.5, "price": 180 },
    { "box_code": 3, "weight": 1, "price": 60 },
    { "kit": "KIT-001" }
  ]
}
```

> In the `kit` box, the weight and dimensions come from the registration of the kit box and the `price` is the sum of the values of the kit items.
> If the registered box has a **maximum weight** configured, the `weight` provided cannot exceed it.

### Package (Box) Quotation — advanced

- **When to use:** it is the **standard flow for shipping** goods. You declare the items and state which box
  each one is in.
- **How to build it:** `object: package` + `type: advanced`. The boxes bring the dimensions; each item points to the
  position of its box in `box_id` (1 = first box).
- The weight and value of each box are calculated as the sum of the items, if they are not provided.
- **Every box needs at least one item**; a `box_id` outside the range of boxes is rejected.
- **It is eligible for shipment.**

#### Minimum payload

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  ]
}
```

#### Maximum payload

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0003",
  "channel_order_number": "334458",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "ior": true,
  "description": "Order with two boxes",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
  "boxes": [
    { "name": "Apparel box", "height": 30, "width": 25, "depth": 15, "weight": 1.2, "price": 120 },
    { "box_code": 3 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "Cotton t-shirt S",
      "description": "100% cotton t-shirt, short sleeve, white",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 4,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true
    },
    {
      "box_id": 2,
      "name": "Sport sneaker",
      "description": "Mens running sneaker",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9
    }
  ],
  "manufacturer": {
    "name": "Fabrica Exemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Warehouse 2",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  }
}
```

> An item with a registered `sku` can be sent with only `box_id`, `sku`, `quantity` and `unit_value`: the rest
> (name, country, HS Code, weight, EAN, `nsmp`) is completed from the [item registration](api-itens-v1.md).
> What you send in the payload takes priority over the registration.

### Pallet Quotation — by LWH

- **When to use:** palletized cargo whose measurements you know — **L**ength (`depth`), **W**idth (`width`)
  and **H**eight (`height`).
- **How to build it:** `object: pallet` + `type: advanced`, with `height`, `width` and `depth` in each pallet and the items
  pointing to the pallet in `box_id`.
- The platform calculates the volume (`L × W × H ÷ 1,000,000` = m³) from the measurements.
- The pallet weight, if omitted, is the sum of the item weights. Provide `weight` to include the weight of the base/packaging.
- `type: simple` is accepted for simulation, but it is **ineligible for shipment**.

#### Minimum payload

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
  "boxes": [
    { "height": 150, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Industrial spare part",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Maximum payload

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0001",
  "channel_order_number": "990011",
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "ior": false,
  "description": "Two pallets of industrial parts",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
  "boxes": [
    { "name": "Pallet 01", "height": 150, "width": 100, "depth": 120, "weight": 280, "price": 900 },
    { "name": "Pallet 02", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Industrial spare part",
      "description": "Steel gear for packaging machine",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Industrial bearing",
      "description": "Steel ball bearing",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

### Pallet Quotation — by m³

- **When to use:** palletized cargo for which you only know the **volume** (cubic measurement), without the individual measurements.
- **How to build it:** `object: pallet` + `type: advanced`, with `volume` in each pallet (m³ in `metric`, ft³ in `imperial`).
- The platform converts the volume into an equivalent cube for pricing:
  `side (cm) = ∛volume (m³) × 100`. E.g.: `1.0` m³ → 100 × 100 × 100 cm; `1.8` m³ → 121.64 × 121.64 × 121.64 cm.
- If you send `volume` **and** the three measurements, the measurements (LWH) take priority.
- You can mix pallets by LWH and by m³ in the same payload (each position of `boxes[]` decides its own option).

#### Minimum payload

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
  "boxes": [
    { "volume": 1.8 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Industrial spare part",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Maximum payload

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0002",
  "channel_order_number": "990012",
  "object": "pallet",
  "type": "advanced",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "ior": false,
  "description": "Pallet by cubic volume and pallet by measurements",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
  "boxes": [
    { "name": "Volume pallet", "volume": 1.8, "weight": 300, "price": 900 },
    { "name": "Measured pallet", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Industrial spare part",
      "description": "Steel gear for packaging machine",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Industrial bearing",
      "description": "Steel ball bearing",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> A pallet without measurements **and** without `volume` is rejected with `422` (validation) or with
> `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³`.

### Items Quotation

> **Exclusive to [Quotation (v2)](api-cotacao-v2.md#items-quotation).** The quotation by items (`type: items`), in which you
> send only the `sku` of the registered items and the platform assembles the boxes, is not part of the v1 contract.
> In v1, always provide the boxes and use [Package — advanced](#package-box-quotation--advanced) with `items[].sku`
> to take advantage of the item registration.

### Quotation with Kit

A **kit** is a set pre-registered in the panel (one box + items with quantities). When you provide
`boxes[].kit`, that position is replaced by the kit box and the kit items are included automatically.

- It can be combined with regular boxes in the same payload.
- If **all** boxes are kits, `items[]` can be omitted.
- In `type: simple`, the kit becomes a box with `price` = sum of the kit items; in `advanced`, the kit items
  enter the quotation pointing to the kit position.

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
  "boxes": [
    { "kit": "KIT-001" }
  ]
}
```

| Status | Message | Cause |
|---|---|---|
| `400` | `O kit :sku é inválido!` | Kit nonexistent, inactive or from another account |
| `400` | `A caixa do kit :sku é inválida!` | The box linked to the kit does not exist or has been deactivated |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Some kit item was removed from the registration |

## Quotations ineligible for shipment

A successfully calculated quotation **cannot always** become a shipment. Before calling `POST /v1/shipping`,
check the conditions below. They are verified in this order at shipment time:

| # | Condition | Status | Message when it fails |
|---|---|---|---|
| 1 | The quotation exists and was created in **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | The quotation belongs to your account | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Account with tax locked to DDP: DDU quotation (`tax: receiver`) without an approved payment | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | The quotation **has not been used yet** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | It is not `type: simple` with `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | The quotation **has not expired** (carrier `valid_until`) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | The chosen carrier is in `carriers[]` of this quotation | `404` | `Contrato especificado não foi listado para esta cotação!` |

> **How to avoid it:** use `advanced` for package/pallet, create the shipment right after the quotation,
> use the `code`/`slug` returned **in this** quotation and make **one quotation per shipment**.
> All shipment validations are in [Shipment (v1)](api-remessa-v1.md#quotations-ineligible-for-shipment).

## Business rules

### Presets applied to the quotation
The [account presets](api-primeiros-passos.md#account-presets) affect the calculation:

| Preset | Effect on the quotation |
|---|---|
| Tax (`tax`) | When configured, it **replaces** the `tax` sent. Destination countries listed as exceptions receive the opposite of the default. If the account is locked to DDP, the tax is always `sender` |
| Automatic insurance | If active, insurance is purchased automatically when the declared value (converted to USD) reaches the configured minimum, even with `insurance: false` |
| Declared freight | Used when `declared_freight` is not sent. Requires the declared freight permission |
| Measurement system | Used when `measurement` is not sent. If it is not configured either, `metric` |
| Residential delivery / non-stackable | Used when `residential_delivery` / `non_stackable` are not sent |
| Sales channel | Used when `channel_code` is not sent |
| Box grouping | Does not apply to v1 (used only in the v2 quotation by items) |

### Address validation
In v1 the manual address is **validated already in the quotation**:

| Case | Status | Message |
|---|---|---|
| Address in Brazil without `postal_code` | `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` |
| Nonexistent/invalid Brazilian ZIP code (when `state_code` is not sent) | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| International address not found | `404` | `Endereço internacional enviado está errado ou não existe!` |
| State/province not recognized | `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |
| `address_code` nonexistent or from another account | `400` | `Nenhum endereco encontrado para o usuário especificado!` |
| Identical sender and receiver | `400` | `Endereço de Remetente e Destinatário são iguais!` |

### Cargo restrictions
The quotation may be rejected when the cargo does not fit the contracts of your account:

| Status | Message | Cause |
|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Box without items in `advanced` |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` greater than the number of boxes or less than 1 |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Manual item without `box_id` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | `weight` above the maximum weight of the registered box (`simple`) |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Sum of the items above the maximum weight of the registered box |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` nonexistent or from another account |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` not registered and incomplete manual data |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...` | `unit_value` equal to or less than zero |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` from another account |
| `404` | `Nenhum transportador encontrado para este usuário!` | Account without a contract enabled for the route/object |
| `404` | `Não há contratos que atendem as especificações do pacote enviado...` | No contract covers the weight/dimension/route combination |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Cargo outside the automatic range — talk to the sales team |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | No carrier returned a result |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item)...` | HS Code without a tax table at destination — review the `hscode` |

> **Weight**-related refusals may come with `status: "error"` instead of `"warning"`, keeping the HTTP code
> from the table. Handle them by the HTTP code and by the `message`.

### Quotation alerts (`warnings`)
The quotation is made, but `data.warnings` appears with:

| Alert | When |
|---|---|
| `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` | Two or more items with the same `name` |
| `Valores declarados dos itens podem ocasionar problemas aduaneiros` | Declared value above USD 800 with an express carrier (DHL/FedEx) in the result |

### Referenced registrations
When you use registration codes, they must exist, be active and belong to your account:

| Field | Registration | Where to manage it |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Box | [Box](api-caixa-v1.md) |
| `items[].sku` | Item | [Items](api-itens-v1.md) |
| `boxes[].kit` | Kit | Shipsmart Global panel |
| `address_*.address_code` / `manufacturer.address_code` | Address | Shipsmart Global panel |
| `channel_code` | Sales channel | Shipsmart Global panel |

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Business rule (cargo, box, item, kit, address) | Read `message` and adjust the scenario |
| `403` | No quotation permission or channel from another account | Ask support for the permission / review `channel_code` |
| `404` | Address not found or no eligible contract | Review the address and scenario |
| `406` | Cargo requires personalized service, or registered sender required for the account type | Talk to the sales team / use `address_sender.address_code` |
| `422` | Field validation or HS Code without a tax table | Fix the fields listed in `data` / review `hscode` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- The `POST` response already brings the calculated result — it is not mandatory to call the `GET` afterwards.
- `data.warnings` is informational: the quotation was made, but there is a customs risk to review.
- For an even leaner quotation payload (country and state only), see [Quotation V2](api-cotacao-v2.md).
- To choose between the versions, see [Quotation & Shipment: v1 vs v2](api-comparativo-cotacao-remessa.md).
- For quotation + shipment in a single synchronous call, see [Manifest](api-manifesto.md).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
