# Quotation (v2)

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
- Same calculation engine as [Quotation v1](api-cotacao-v1.md), with a **much leaner address contract**.
- Use v2 when, at quotation time, you still **do not have** the receiver's city and postal code — only the country
  (and optionally the state).
- The city, the postal code and the full address are informed later, in [Shipment V2](api-remessa-v2.md).

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
1. `POST /v2/quotation` informing only the country (and the state, if you know it) for the origin and the destination.
2. The response already brings the calculated result (`data.carriers`) and the `data.quotation` code.
3. Choose a carrier and create the [Shipment V2](api-remessa-v2.md) with the full address,
   **before** `valid_until`.

```
POST /v2/quotation  ──►  quotation + carriers[]
  (country/state only)         │
                               ▼
                    POST /v2/shipping
              (full address + carrier_code|slug)
```

## Differences from v1

| Topic | v1 | v2 |
|---|---|---|
| Address in the quotation | `country_code` + `city` + (`postal_code` or `state_code`/`state`) | Only `country_code` (+ optional `state_code`) |
| Address in the shipment | Street, name, e‑mail, phone, type | Street, name, e‑mail, phone, type **+ city, state and postal code** |
| Querying the result | `GET /v1/quotation/{quotation}/{level}` | **Does not exist.** The result comes in the `POST` itself |
| `ior` (Importer of Record) | Supported | **Not part of the v2 contract** |
| `items[].selling_unit_value` | Supported | **Not part of the v2 contract** |
| Calculation accuracy | Higher: city/postal code allow regional fees to be applied already in the quotation | The amount may be refined in the shipment, when the full address is informed |

> Everything else (object, type, boxes, items, kits, currencies, declared freight, manufacturer, warnings)
> is identical to v1.

## Essential concepts

### `object` — what is being shipped
| Value | Meaning | Notes |
|---|---|---|
| `doc` | Document | Quotation per box, informing `weight` and `price` |
| `package` | Package/box | Most common flow |
| `pallet` | Pallet | Allows informing `volume` instead of height/width/depth |
| `not_doc` | Legacy | Internally converted to `package`. **Do not use it in new integrations** |

### `type` — how the calculation is assembled
| Value | What you send | When to use it |
|---|---|---|
| `simple` | Only `boxes[]` with `weight` and `price` | Quick simulation. **Not eligible for a shipment when `object` is `package` or `pallet`** |
| `advanced` | `boxes[]` (dimensions) + `items[]` with `box_id` | Standard flow for a shipment |
| `items` | Only `items[]` with `sku` | When items and boxes are already registered and the platform does the fitting |

### `measurement` — measurement system
| Value | Dimensions | Weight | Volume |
|---|---|---|---|
| `metric` (default) | centimeters | kilograms | m³ |
| `imperial` | inches | pounds | ft³ |

### `level` — level of detail of the result
| Value | Return |
|---|---|
| `simple` (default) | Lean list of carriers with totals |
| `detailed` | Includes the detailed breakdown (freight, insurance, tax, before and after the administrative fees) |

Sent as a **query string**: `POST /v2/quotation?level=detailed`.

## Endpoint: create quotation

- **Method and route:** `POST /v2/quotation`
- **Optional query string:** `level=simple|detailed`

### Parameters — payload root

| Field | Type | Required | Min. | Max. | Default | Accepted values | Description |
|---|---|---|---|---|---|---|---|
| `object` | string | **yes** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | What is being shipped |
| `type` | string | **yes** | - | - | - | `simple`, `advanced`, `items` | How the calculation is assembled |
| `tax` | string | **yes** | - | - | - | `sender`, `receiver` | Who pays the taxes |
| `insurance` | boolean | **yes** | - | - | - | `true`, `false` | Purchase insurance |
| `currency_quote` | string | **yes** | 3 | 3 | - | see [currencies](api-primeiros-passos.md#accepted-currencies) | Currency in which the quotation is presented |
| `currency_payment` | string | **yes** | 3 | 3 | - | see [currencies](api-primeiros-passos.md#accepted-currencies) | Currency in which the payment will be made |
| `address_sender` | object | **yes** | - | - | - | - | Origin (see table below) |
| `address_receiver` | object | **yes** | - | - | - | - | Destination (see table below) |
| `boxes` | array | conditional | - | - | - | - | Required when `type` is **not** `items` |
| `items` | array | conditional | - | - | - | - | Required when `type` is **not** `simple` and no box uses `kit` |
| `measurement` | string | no | - | - | `metric` | `metric`, `imperial` | Measurement system |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Delivery to a residential address |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Non-stackable packages |
| `description` | string | no | - | 155 | `null` | - | Free description of the quotation |
| `channel_code` | integer | no | - | - | account preset | - | Code of the registered sales channel |
| `channel_identifier_number` | string | no | - | - | `null` | - | Your commercial identifier (order no.). **Recommended** |
| `channel_order_number` | string | no | - | 100 | `null` | - | Order number in the sales channel |
| `declared_freight` | object | no | - | - | account preset | - | Declared freight (see table below) |
| `manufacturer` | object | no | - | - | `null` | - | Manufacturer shown on the Commercial Invoice (see table below) |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Level of detail of the result |

### Parameters — `address_sender` and `address_receiver`

**This is the main difference in v2.** Choose one of the two forms:

**Form A — address already registered:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this form | Code of an address registered in your account |

**Form B — only the country (and the state, if you know it):**

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `country_code` | string | **yes** | 2 | 2 | Country in ISO 3166-1 alpha-2 (e.g. `BR`, `US`) |
| `state_code` | string | no | - | - | State/province abbreviation (e.g. `SP`, `FL`). When informed, it improves the accuracy of the regional tax calculation |

> Do not send `city`, `postal_code` or `state` in the v2 quotation — this data is informed in the shipment.

### Parameters — `boxes[]`

Each element of the array represents one package. There are **three ways** to describe the box:

**Form A — registered kit** (fills in the box **and** the items automatically):

| Field | Type | Required | Max. | Description |
|---|---|---|---|---|
| `kit` | string | **yes** in this form | 50 | SKU of the registered kit. The other box fields and the items of that position are filled in from the kit registration |

**Form B — registered box:**

| Field | Type | Required | Description |
|---|---|---|---|
| `box_code` | integer | **yes** in this form | Code of the registered box ([Box](api-caixa-v1.md)) |
| `box_identifier` | string | alternative to `box_code` | Identifier of the registered box (e.g. `CX0236`) |

**Form C — box informed manually:**

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `name` | string | no | 4 | 60 | Box name/label |
| `height` | number | **yes** | 1 | - | Height |
| `width` | number | **yes** | 1 | - | Width |
| `depth` | number | **yes** | 1 | - | Depth |
| `volume` | number | conditional | 0.001 | - | **Only when `object` is `pallet`.** Alternative to height/width/depth |

**Fields valid in any form (except `kit`):**

| Field | Type | Required | Min. | Description |
|---|---|---|---|---|
| `weight` | number | **yes** when `object` is `doc`/`package` **and** `type` is `simple` | > 0 | Package weight |
| `price` | number | **yes** when `object` is `doc`/`package` **and** `type` is `simple` | > 0 | Declared value of the package |

### Parameters — `items[]`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `quantity` | integer | **yes** | 1 | 1000 | Item quantity |
| `unit_value` | number | **yes** | - | - | Unit value in the quotation currency |
| `box_id` | integer | **yes**, except when `type` is `items` | - | - | **Position** of the box in the `boxes` array (1 = first box). It is not the `box_code` |
| `sku` | string | **yes** when `type` is `items` | - | 50 | SKU of the registered item ([Items](api-itens-v1.md)). When informed, the registered data is used |
| `ean` | string | no | - | 50 | EAN code of the item |
| `name` | string | **yes** when `sku` is not informed | 6 | 50 | Item name (appears on the invoice) |
| `description` | string | no | - | 155 | Detailed description of the item |
| `country_code` | string | **yes** when `sku` is not informed | 2 | 2 | Country of origin/manufacture of the item |
| `hscode` | string | **yes** when `sku` is not informed | 6 | 10 | NCM/HS Code. Defines the applied tax table |
| `weight` | number | **yes** when `sku` is not informed | 0.001 | - | Unit weight of the item |
| `is_groupable` | boolean | no | - | - | Default `true`. When `false`, the item gets an exclusive box in the `smartbox` grouping |

### Parameters — `declared_freight`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Declared freight value |
| `currency` | string | **yes** when `value` is informed | 3 | 3 | Currency of the declared freight |

### Parameters — `manufacturer`

Manufacturer data printed on the Commercial Invoice. Send `address_code` **or** the manual fields.

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `address_code` | integer | alternative to the manual fields | - | - | Code of a registered address |
| `name` | string | **yes** without `address_code` | 5 | 30 | Legal/trade name of the manufacturer |
| `country_code` | string | **yes** without `address_code` | 2 | 2 | Country |
| `state_code` | string | **yes** without `address_code` | 1 | 10 | State abbreviation |
| `state` | string | no | - | 155 | State name |
| `postal_code` | string | **yes** without `address_code` | - | 12 | Postal/ZIP code |
| `city` | string | **yes** without `address_code` | - | 100 | City |
| `street` | string | **yes** without `address_code` | - | 70 | Street |
| `number` | string | no | - | 6 | Number |
| `district` | string | no | - | 30 | District |
| `complement` | string | no | - | 20 | Complement |
| `email` | string | **yes** without `address_code` | - | 60 | E-mail |
| `phone` | string | **yes** without `address_code` | - | 25 | Phone |
| `phone_extension` | string | no | - | 6 | Extension |
| `type` | string | **yes** without `address_code` | - | - | `pf` or `pj` |
| `federal_tax_id` | string | no | - | 20 | CPF/CNPJ or equivalent |
| `state_tax_id` | string | no | - | 20 | State tax registration |

### Minimum payload

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 0.5,
      "price": 100
    }
  ]
}
```

### Minimum payload eligible for a shipment

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
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

### Maximum payload

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "channel_order_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
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
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Cotton t-shirt S",
      "description": "100% cotton t-shirt, short sleeve, white",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
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
        "url_image": "https://example.s3.amazonaws.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "real", "weight": "5" },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "insurance": "11.55",
        "insurance_final": "11.55",
        "tax": "24924.09",
        "tax_final": "24924.09",
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `data.quotation` | string (UUID) | **Quotation code.** Use it in `POST /v2/shipping` |
| `data.finished` | boolean | Indicates whether the quotation processing was completed |
| `data.warnings` | array | **Optional.** Only appears when there are customs alerts |
| `carriers[].code` | integer | Contract code. Use it in `carrier_code` in the shipment |
| `carriers[].slug` | string | Text identifier of the contract. Alternative to `carrier_code` |
| `carriers[].valid_until` | string | **Quotation validity.** Create the shipment before this date/time |
| `carriers[].transit_days` | integer | Estimated transit time in days |
| `carriers[].weight_details.type` | string | `real` or `cubed` — which weight was used in the rating |
| `carriers[].freight` / `freight_final` | string | Freight before / after the administrative fees |
| `carriers[].insurance` / `insurance_final` | string | Insurance before / after the administrative fees |
| `carriers[].tax` / `tax_final` | string | Taxes before / after the administrative fees |
| `carriers[].currency_payment_amount` | string | Total payable in the payment currency |

## Business rules

### Eligibility for a shipment
1. The `quotation` exists and belongs to your account.
2. The quotation **is not expired** (respect `valid_until`).
3. The quotation **has not been used yet** in another shipment.
4. The quotation was created in **v2** and will be used in `POST /v2/shipping`.
5. It is not a `simple` quotation of `package`/`pallet`.
6. The informed carrier is in the list returned by the quotation.

### Cargo restrictions
The same restrictions as in v1 apply (dimensions, weight per package, total weight, item limit,
contract availability, blocked route, HS Code without a tax table).
See the **Cargo restrictions** section in [Quotation (v1)](api-cotacao-v1.md#cargo-restrictions).

### No query endpoint
v2 **does not have** `GET /v2/quotation/{quotation}`. The complete result is delivered in the `POST` response,
so **store the response** if you need to present it to the user again.

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Business rule (cargo restriction, route, contract, HS Code) | Read `message` and adjust the scenario |
| `403` | No quotation permission | Ask support for the permission |
| `406` | Incompatible payload structure | Review types and nesting |
| `422` | Field validation | Fix the fields listed in `data` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- v2 trades accuracy for simplicity: the full address only comes in with the shipment.
- Operational flows for **status, documents, cancellation and pickup scheduling** use the same
  `/v1/...` endpoints, even for shipments created in v2 — see
  [Tracking](api-rastreio.md), [Documents](api-documentos.md) and [Scheduling](api-agendamento.md).
- Store the `POST` response: there is no re-query endpoint in v2.
