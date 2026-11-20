# Quotation (v2)

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Same calculation as [Quotation v1](api-cotacao-v1.md), with a **much leaner address contract**.
- Use v2 when, at quotation time, you **do not have** the city and ZIP code yet — only the country
  (and optionally the state).
- The city, ZIP code and full address are provided and validated later, in [Shipment V2](api-remessa-v2.md).
- It covers the v1 cargo scenarios **and** the **quotation by items** (`type: items`), **exclusive to v2**: you send
  only the `sku` of the registered items, without providing boxes — see [Quotation scenarios](#quotation-scenarios).

> **Important:** a quotation created in v2 can only generate a shipment in v2 (`POST /v2/shipping`).
> See the [comparison](api-comparativo-cotacao-remessa.md) to choose the version.

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
2. `POST /v2/quotation` providing only the country (and the state, if you know it) for origin and destination.
3. The response already brings the calculated result (`data.carriers`) and the `data.quotation` code. **Store the response**:
   there is no re-query endpoint in v2.
4. Check whether the quotation is [eligible for shipment](#quotations-ineligible-for-shipment).
5. Choose a carrier and create the [Shipment V2](api-remessa-v2.md) with the full address,
   **before** the `valid_until`.

```
POST /v2/quotation  ──►  quotation + carriers[]
 (country/state only)          │
                               ▼
                    POST /v2/shipping
              (full address + carrier_code|slug)
```

## Differences from v1

| Topic | v1 | v2 |
|---|---|---|
| Address in the quotation | `country_code` + `city` + (`postal_code` or `state_code`/`state`) | Only `country_code` (+ optional `state_code`) |
| Quotation by items (`type: items`) | **Not available**: the boxes are always provided | **Available**: only the `sku` of the registered items, no boxes |
| Address validation | In the quotation (ZIP code, city, state) | In the shipment, when the full address is sent |
| Address in the shipment | Street, name, email, phone, type | Street, name, email, phone, type **+ city, state and ZIP code** |
| Result retrieval | `GET /v1/quotation/{quotation}/{level}` | **Does not exist.** The result comes in the `POST` itself |
| `ior` (Importer of Record) | Supported | **Not part of the v2 contract** (ignored) |
| `items[].selling_unit_value` | Supported | **Not part of the v2 contract** (ignored) |
| Calculation accuracy | Higher: city/ZIP code allow regional fees to be applied already in the quotation | The value considers country/state; the full address comes in with the shipment |

> These are the two reasons v2 exists: **reduced address** (country/state only) and **quotation by items**.
> Everything else (object, boxes, kits, currencies, declared freight, manufacturer, warnings, account presets) is identical to v1.

## Key concepts

### `object` — what is being shipped
| Value | Meaning | Notes |
|---|---|---|
| `doc` | Document | Quoted by volume, providing the `weight` and `price` of each envelope/box |
| `package` | Package/box | Most common flow |
| `pallet` | Pallet | Accepts dimensions (LWH) **or** `volume` in m³/ft³ |
| `not_doc` | Legacy | Converted internally to `package`. **Do not use it in new integrations** |

### `type` — how the calculation is built
| Value | What you send | When to use |
|---|---|---|
| `simple` | Only `boxes[]` with `weight` and `price` | Documents and quick simulations. **Not eligible for shipment when `object` is `package` or `pallet`** |
| `advanced` | `boxes[]` (dimensions) + `items[]` with `box_id` | Standard flow for shipment |
| `items` | `items[]` with `sku` (and, depending on the account grouping, candidate `boxes[]`) | **Exclusive to v2.** When the items are already registered and the platform assembles the boxes |

### Eligibility matrix

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Quotation and shipment | Not recommended | Not recommended |
| `package` | ⚠️ Simulation only (ineligible for shipment) | ✅ Quotation and shipment | ✅ Quotation and shipment |
| `pallet` | ⚠️ Simulation only (ineligible for shipment) | ✅ Quotation and shipment | ✅ Quotation and shipment |

### `tax` — who pays the taxes
| Value | Meaning |
|---|---|
| `sender` | DDP: the sender pays taxes and fees at destination |
| `receiver` | DDU: the receiver pays taxes and fees at destination |

> If your account has a **tax preset** configured, it **replaces** the value sent in `tax`.
> See [Presets applied to the quotation](api-cotacao-v1.md#presets-applied-to-the-quotation).

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

Sent as a **query string**: `POST /v2/quotation?level=detailed`.

## Endpoint: create quotation

- **Method and route:** `POST /v2/quotation`
- **Optional query string:** `level=simple|detailed`
- **When to use:** when, at quotation time, you only know the country (and perhaps the state) of origin and destination.

### Parameters — payload root

| Field | Type | Required | Min. | Max. | Default | Accepted values | Description |
|---|---|---|---|---|---|---|---|
| `object` | string | **yes** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | What is being shipped |
| `type` | string | **yes** | - | - | - | `simple`, `advanced`, `items` | How the calculation is built (`items` only exists in v2) |
| `tax` | string | **yes** | - | - | - | `sender`, `receiver` | Who pays the taxes (may be replaced by the account preset) |
| `insurance` | boolean | **yes** | - | - | - | `true`, `false` | Purchase insurance (may be enabled automatically by the account) |
| `currency_quote` | string | **yes** | 3 | 3 | - | see [currencies](api-primeiros-passos.md#accepted-currencies) | Currency in which the quotation is presented and in which `unit_value`/`price` are provided |
| `currency_payment` | string | **yes** | 3 | 3 | - | see [currencies](api-primeiros-passos.md#accepted-currencies) | Currency in which the payment will be made |
| `address_sender` | object | **yes** | - | - | - | - | Origin (see table below) |
| `address_receiver` | object | **yes** | - | - | - | - | Destination (see table below) |
| `boxes` | array | conditional | - | - | - | - | Required when `type` is **not** `items`. In `type: items` with `smartbox` grouping, it is also required |
| `items` | array | conditional | - | - | - | - | Required when `type` is **not** `simple` and no box uses `kit` |
| `measurement` | string | no | - | - | account preset or `metric` | `metric`, `imperial` | Measurement system |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Delivery to a residential address |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Non-stackable packages |
| `description` | string | no | - | 155 | `null` | - | Free description of the quotation |
| `channel_code` | integer | no | - | - | account preset | - | Code of the sales channel registered and enabled for your user |
| `channel_identifier_number` | string | no | - | - | `null` | - | Your commercial identifier (order no.). **Recommended** |
| `channel_order_number` | string | no | - | 100 | `null` | - | Order number in the sales channel |
| `declared_freight` | object | no | - | - | account preset | - | Declared freight. Requires a specific permission |
| `manufacturer` | object | no | - | - | `null` | - | Manufacturer shown on the Commercial Invoice |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Level of detail of the result |

### Parameters — `address_sender` and `address_receiver`

**This is the main difference in v2.** Choose one of the two options and do not mix them:

**Option A — already registered address:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this option | Code of an address registered in your account panel |

**Option B — country only (and the state, if you know it):**

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `country_code` | string | **yes** | 2 | 2 | Country in ISO 3166-1 alpha-2 (e.g. `BR`, `US`) |
| `state_code` | string | no | - | - | State/province abbreviation (e.g. `SP`, `FL`). When provided, it improves the accuracy of the regional tax calculation and is reused in the shipment |

> Do not send `city`, `postal_code` or `state` in the v2 quotation: these fields are **ignored** and must be
> provided in [Shipment V2](api-remessa-v2.md).

> **Registered sender required for some account types:** individual accounts and business accounts
> with a single distribution center must provide the sender using **Option A** (`address_code`).
> Without it, the API responds `406` with `Codigo de endereço do remetente não encontrado!`.

### Parameters — `boxes[]`

Identical to v1. There are **three ways** to describe the package:

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
| `name` | string | no | 4 | 60 | Name/label of the package |
| `height` | number | **yes** (except pallet with `volume`) | 1 | - | Height |
| `width` | number | **yes** (except pallet with `volume`) | 1 | - | Width |
| `depth` | number | **yes** (except pallet with `volume`) | 1 | - | Depth (length) |
| `volume` | number | conditional | 0.001 | - | **Only when `object` is `pallet`.** Alternative to height/width/depth, in m³ or ft³ |

**Fields valid in options B and C:**

| Field | Type | Required | Min. | Description |
|---|---|---|---|---|
| `weight` | number | **yes** when `object` is `doc`/`package` **and** `type` is `simple` | > 0 | Weight of the package. In `advanced`/`items`, when omitted, it is the sum of the items |
| `price` | number | **yes** when `object` is `doc`/`package` **and** `type` is `simple` | > 0 | Declared value of the package. In `advanced`/`items`, when omitted, it is the sum of the items |

### Parameters — `items[]`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `quantity` | integer | **yes** | 1 | 1000 | Quantity of the item |
| `unit_value` | number | **yes** | > 0 | - | Unit value in the quotation currency. Zero or negative is rejected |
| `box_id` | integer | **yes**, except when `type` is `items` | 1 | no. of boxes | **Position** of the box in the `boxes` array (1 = first box). It is not the `box_code` |
| `sku` | string | **yes** when `type` is `items` | - | 50 | SKU of the registered item ([Items](api-itens-v1.md)). Omitted fields are completed from the registration |
| `ean` | string | no | - | 50 | EAN code of the item |
| `nsmp` | string | no | - | 50 | *Non Standard Manufacturer Product Id*: the manufacturer's product identifier. Printed on the invoice and used in the regulatory information required by some carriers for destinations in the European Union |
| `name` | string | **yes** when `sku` is not provided | 6 | 50 | Item name (shown on the invoice) |
| `description` | string | no | - | 155 | Detailed description of the item |
| `country_code` | string | **yes** when `sku` is not provided | 2 | 2 | Country of origin/manufacture of the item |
| `hscode` | string | **yes** when `sku` is not provided | 6 | 10 | NCM/HS Code. Defines the tax table applied |
| `weight` | number | **yes** when `sku` is not provided | 0.001 | - | Unit weight of the item |
| `is_groupable` | boolean | no | - | - | Default `true`. When `false`, the item receives an exclusive box in the `type: items` groupings |

### Parameters — `declared_freight`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Declared freight value |
| `currency` | string | **yes** when `value` is provided | 3 | 3 | Currency of the declared freight |

> Applied only if your profile has the **declared freight permission**; otherwise it is silently ignored.

### Parameters — `manufacturer`

Identical to v1: send `address_code` **or** the manual fields (`name`, `country_code`, `state_code`, `state`,
`postal_code`, `city`, `street`, `number`, `district`, `complement`, `email`, `phone`, `phone_extension`, `type`,
`federal_tax_id`, `state_tax_id`). Limits and recommendations in the
[v1 manufacturer table](api-cotacao-v1.md#parameters--manufacturer).

| Field | Type | Recommended | Description |
|---|---|---|---|
| `address_code` | integer | alternative to the manual fields | Registered manufacturer address |
| `name` | string | **yes** without `address_code` | Company name/manufacturer name (5 to 30 characters) |

### Minimum payload

The smallest payload accepted by the endpoint (document quotation):

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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
  "description": "Marketplace order, october batch",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    { "name": "Master box", "height": 20, "width": 20, "depth": 20, "weight": 2.5, "price": 150 },
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
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

### Request example

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v2/quotation?level=detailed' \
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
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
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
        "weight_details": { "type": "real", "weight": "5" },
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
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "url_image": "https://files.example.com/contratos/fedex.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.55",
        "insurance_final": "11.55",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "24924.09",
        "tax_final": "24924.09",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Response fields

Same structure as v1 — full description in [v1 response fields](api-cotacao-v1.md#response-fields).
The most used fields:

| Field | Type | Description |
|---|---|---|
| `data.quotation` | string (UUID) | **Quotation code.** Use it in `POST /v2/shipping` |
| `data.finished` | boolean | `true` when the quotation **has already been used** to create a shipment |
| `data.warnings` | array | **Optional.** Only appears when there are customs alerts |
| `carriers[].code` | integer | Contract code. Use it in `carrier_code` in the shipment |
| `carriers[].slug` | string | Text identifier of the contract. Alternative to `carrier_code` |
| `carriers[].valid_until` | string | **Quotation validity.** Create the shipment before this date/time |
| `carriers[].transit_days` | integer | Estimated transit time in days |
| `carriers[].weight_details.type` | string | `real` or `cubed` — which weight was used for pricing |
| `carriers[].freight` / `freight_final` | string | Freight before / after administrative fees |
| `carriers[].insurance` / `insurance_final` | string | Insurance before / after administrative fees |
| `carriers[].tax` / `tax_final` | string | Taxes before / after administrative fees |
| `carriers[].currency_quote_amount` | string | Total to pay in the quotation currency |
| `carriers[].currency_payment_amount` | string | Total to pay in the payment currency |

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
| [Items](#items-quotation) — **exclusive to v2** | `package` | `items` | Depends on the account grouping | Required, with `sku` | ✅ |

### Document Quotation

- **When to use:** shipping papers/documents with no relevant commercial value (contracts, certificates, printed matter).
- **How to build it:** `object: doc` + `type: simple`. Each package (envelope/box) provides dimensions, `weight` and `price`.
- **Does not use** `items[]` or `manufacturer`: documents do not go through the per-item tax calculation.
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
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
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
  [Package — advanced](#package-box-quotation--advanced) or [Items](#items-quotation).

#### Minimum payload

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
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
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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
  "description": "Order with two boxes",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
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
      "unit_value": 199.9
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
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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
  "description": "Two pallets of industrial parts",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
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
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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
  "description": "Pallet by cubic volume and pallet by measurements",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
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

- **Exclusive to v2.** It is one of the reasons to choose v2: the payload does not need to describe the package (box).
- **When to use:** you already have the products in the [item registration](api-itens-v1.md) and want the platform
  to assemble the boxes on its own.
- **How to build it:** `object: package` + `type: items`. Each item provides `sku`, `quantity` and `unit_value`; do **not**
  provide `box_id`.
- How the boxes are assembled depends on the **grouping** configured in your account
  ([Getting Started](api-primeiros-passos.md#box-grouping)):

| Grouping | `boxes[]` in the payload | Prerequisite in the item registration | How the boxes are assembled |
|---|---|---|---|
| `default` | Not used | Item linked to a registered box (`box_code` or `box_identifier`) | Groupable items that use the same box are combined; non-groupable items receive one box per unit |
| `all_in_one` | Not used | Item with registered dimensions | Groupable items go into **one** virtual box with the total volume; non-groupable items receive one box per unit |
| `smartbox` | **Required** (candidate boxes) | Item with registered dimensions | The items are fitted into the boxes sent, considering volume and dimensions with 10% slack and `is_groupable`/`is_foldable` |

#### Minimum payload

`default` or `all_in_one` grouping:

```json
{
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "items": [
    { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
  ]
}
```

#### Maximum payload

`smartbox` grouping (candidate boxes provided):

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0004",
  "channel_order_number": "334459",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Order assembled from the item registration",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 25, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "name": "Large box", "height": 40, "width": 40, "depth": 40 }
  ],
  "items": [
    {
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
      "is_groupable": true
    },
    {
      "sku": "TEN-042-M",
      "quantity": 1,
      "unit_value": 199.9,
      "is_groupable": false
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> Typical errors in this scenario: `O item com o sku :name não existe!` (`404`),
> `A caixa com o box_code/box_identifier :name não existe ou está desativada` (`404`),
> `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (`400`) and
> `Nenhuma caixa comporta o item :name (Smart Box).` (`400`).

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
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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

The same conditions as v1 apply to v2, verified in `POST /v2/shipping` in this order:

| # | Condition | Status | Message when it fails |
|---|---|---|---|
| 1 | The quotation exists and was created in **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | The quotation belongs to your account | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Account with tax locked to DDP: DDU quotation (`tax: receiver`) without an approved payment | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | The quotation **has not been used yet** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | It is not `type: simple` with `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | The quotation **has not expired** (carrier `valid_until`) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | The chosen carrier is in `carriers[]` of this quotation | `404` | `Contrato especificado não foi listado para esta cotação!` |

> In v2 the full address is only validated in the shipment. An invalid ZIP code or address provided in
> [Shipment V2](api-remessa-v2.md#full-address-validation) also prevents the creation.

## Business rules

### Presets and cargo restrictions
The same rules as v1 apply:
[Presets applied to the quotation](api-cotacao-v1.md#presets-applied-to-the-quotation),
[Cargo restrictions](api-cotacao-v1.md#cargo-restrictions),
[Quotation alerts](api-cotacao-v1.md#quotation-alerts-warnings) and
[Referenced registrations](api-cotacao-v1.md#referenced-registrations).

### Address without validation in the quotation
- In v2, the quotation does **not** validate ZIP code, city or state; only the country (and the `state_code`, when sent).
- There is no ZIP code requirement for addresses in Brazil in the v2 quotation.
- The `state_code` provided in the quotation is reused in the v2 shipment.

### No retrieval endpoint
v2 **does not have** `GET /v2/quotation/{quotation}`. The full result is delivered in the `POST` response,
so **store the response** if you need to present it to the user again.

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Business rule (cargo, box, item, kit) | Read `message` and adjust the scenario |
| `403` | No quotation permission or channel from another account | Ask support for the permission / review `channel_code` |
| `404` | Item/box not found in the grouping or no eligible contract | Review registrations and scenario |
| `406` | Cargo requires personalized service, or registered sender required for the account type | Talk to the sales team / use `address_sender.address_code` |
| `422` | Field validation or HS Code without a tax table | Fix the fields listed in `data` / review `hscode` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- v2 trades accuracy for simplicity: the full address only comes in with the shipment.
- The operational flows for **status, documents, cancellation and pickup scheduling** use the same
  `/v1/...` endpoints, even for shipments created in v2 — see
  [Tracking](api-rastreio.md), [Documents](api-documentos.md) and [Scheduling](api-agendamento.md).
- Store the `POST` response: there is no re-query endpoint in v2.
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
