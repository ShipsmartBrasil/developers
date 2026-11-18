# Quotation (v1)

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
- Calculates freight, insurance and taxes for an international shipment, across all carriers enabled for your account.
- It is the **mandatory first step** of the flow: the shipment, the tracking, the documents and the pickup scheduling
  are all tied to the quotation code (`quotation`).
- v1 is the complete version: it accepts a detailed origin/destination address and has an endpoint to re-query the result.

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
1. `POST /v1/quotation` with origin, destination, object, type and boxes/items.
2. The response already brings the calculated result (`data.carriers`) and the `data.quotation` code.
3. If you need to re-query (or change the level of detail), use `GET /v1/quotation/{quotation}/{level}`.
4. Choose a carrier from the list and use `code` (or `slug`) + `quotation` to create the
   [shipment](api-remessa-v1.md) **before** `valid_until`.

```
POST /v1/quotation  ──►  quotation + carriers[]
                              │
                              ▼
                   POST /v1/shipping  (carrier_code | slug)
```

## Essential concepts

### `object` — what is being shipped
| Value | Meaning | Notes |
|---|---|---|
| `doc` | Document | Quotation per box, informing `weight` and `price` |
| `package` | Package/box | Most common flow |
| `pallet` | Pallet | Allows informing `volume` instead of height/width/depth |
| `not_doc` | Legacy | Kept for compatibility; it is internally converted to `package`. **Do not use it in new integrations** |

### `type` — how the calculation is assembled
| Value | What you send | When to use it |
|---|---|---|
| `simple` | Only `boxes[]` with `weight` and `price` | Quick freight simulation. **Not eligible for a shipment when `object` is `package` or `pallet`** |
| `advanced` | `boxes[]` (dimensions) + `items[]` with `box_id` pointing to the box position | Standard flow for a shipment: you control the distribution of the items across the boxes |
| `items` | Only `items[]` with `sku` (the boxes are chosen by the platform) | When the items and the boxes are already registered and you want the platform to do the fitting |

> **Attention:** `type: simple` with `object: package` or `object: pallet` is only meant for simulation.
> When you try to create the shipment, the API responds `Cotação Simples é inelegível para remessa!`.
> For a shipment use `advanced` or `items`.

### `tax` — who pays the taxes
| Value | Meaning |
|---|---|
| `sender` | DDP: the sender pays taxes and fees at the destination |
| `receiver` | DDU: the receiver pays taxes and fees at the destination |

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

Sent as a **query string** in the `POST` and as a **path segment** in the `GET`.

## Endpoint: create quotation

- **Method and route:** `POST /v1/quotation`
- **Optional query string:** `level=simple|detailed`
- **When to use:** whenever you need price, transit time and taxes, and before creating any shipment.

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
| `ior` | boolean | no | - | - | `false` | `true`, `false` | Importer of Record. When `true`, `items[].selling_unit_value` becomes necessary |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Delivery to a residential address |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Non-stackable packages |
| `description` | string | no | - | 155 | `null` | - | Free description of the quotation, for your own reference |
| `channel_code` | integer | no | - | - | account preset | - | Code of the sales channel registered in your account |
| `channel_identifier_number` | string | no | - | - | `null` | - | Your commercial identifier (order no.). **Recommended**: allows querying status/documents by channel |
| `channel_order_number` | string | no | - | 100 | `null` | - | Order number in the sales channel |
| `declared_freight` | object | no | - | - | account preset | - | Declared freight (see table below) |
| `manufacturer` | object | no | - | - | `null` | - | Manufacturer shown on the Commercial Invoice (see table below) |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Level of detail of the result |

### Parameters — `address_sender` and `address_receiver`

There are **two ways** to inform the address. Choose one and do not mix them:

**Form A — address already registered:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this form | Code of an address registered in your account. When used, do not send the other fields |

**Form B — manual address:**

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `country_code` | string | **yes** | 2 | 2 | Country in ISO 3166-1 alpha-2 (e.g. `BR`, `US`) |
| `city` | string | **yes** | - | 100 | City |
| `postal_code` | string | conditional | - | 12 | Postal/ZIP code. Required if `state` is not informed. **For addresses in Brazil it is always required** |
| `state` | string | conditional | - | - | Name of the state/province. Required if `state_code` and `postal_code` are not informed |
| `state_code` | string | conditional | - | 10 | State/province abbreviation (e.g. `SP`, `FL`). Required if `state` and `postal_code` are not informed |

> **Rule of thumb:** always inform `country_code` + `city` + (`postal_code` **or** `state_code`).
> This combination covers every country.

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

> `weight` and `price` are **ignored** when the box uses `kit`.

### Parameters — `items[]`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `quantity` | integer | **yes** | 1 | 1000 | Item quantity |
| `unit_value` | number | **yes** | - | - | Unit value in the quotation currency |
| `box_id` | integer | **yes**, except when `type` is `items` | - | - | **Position** of the box in the `boxes` array (1 = first box, 2 = second...). It is not the `box_code` |
| `sku` | string | **yes** when `type` is `items` | - | 50 | SKU of the registered item ([Items](api-itens-v1.md)). When informed, the registered data is used and the other fields can be omitted |
| `ean` | string | no | - | 50 | EAN code of the item |
| `name` | string | **yes** when `sku` is not informed | 6 | 50 | Item name (appears on the invoice) |
| `description` | string | no | - | 155 | Detailed description of the item |
| `country_code` | string | **yes** when `sku` is not informed | 2 | 2 | Country of origin/manufacture of the item |
| `hscode` | string | **yes** when `sku` is not informed | 6 | 10 | NCM/HS Code of the item. Defines the applied tax table |
| `weight` | number | **yes** when `sku` is not informed | 0.001 | - | Unit weight of the item |
| `selling_unit_value` | number | conditional | - | - | Unit selling value. **Necessary when `ior` is `true`** |
| `is_groupable` | boolean | no | - | - | Default `true`. When `false`, the item gets an exclusive box in the `smartbox` grouping |

> **Identical descriptions on different items** generate a `warning` in the response
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> The quotation is performed, but it is worth reviewing the descriptions to avoid being held at customs.

### Parameters — `declared_freight`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Declared freight value |
| `currency` | string | **yes** when `value` is informed | 3 | 3 | Currency of the declared freight. Same list as `currency_quote` |

### Parameters — `manufacturer`

Manufacturer data printed on the Commercial Invoice. Send `address_code` **or** the manual fields.

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `address_code` | integer | alternative to the manual fields | - | - | Code of a registered address. When used, the other fields are optional |
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

> For items with an `hscode` starting with `64` (footwear), the manufacturer name is required by customs.

### Minimum payload

Document quotation, simple type — the smallest accepted payload:

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
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
    "state_code": "FL",
    "postal_code": "33101"
  },
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

To generate a package shipment you must use `advanced` (or `items`) and declare the items:

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
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
  ]
}
```

### Maximum payload

All fields accepted by the endpoint:

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
      "description": "Men's running sneaker",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9,
      "is_groupable": false
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
        "url_image": "https://example.s3.amazonaws.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
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
| `data.finished` | boolean | Indicates whether the quotation processing was completed |
| `data.date` | string | Quotation date/time |
| `data.channel` | string | Name of the sales channel, when `channel_code` was informed |
| `data.warnings` | array | **Optional.** Only appears when there are alerts (e.g. duplicate descriptions, high declared values) |
| `data.carriers[]` | array | Calculated shipping options |
| `carriers[].code` | integer | Contract code. Use it in `carrier_code` in the shipment |
| `carriers[].slug` | string | Text identifier of the contract. Alternative to `carrier_code` in the shipment |
| `carriers[].name` | string | Trade name of the carrier |
| `carriers[].url_image` | string | Carrier logo |
| `carriers[].valid_until` | string | **Quotation validity.** The shipment must be created before this date/time |
| `carriers[].transit_days` | integer | Estimated transit time in days |
| `carriers[].weight_details.type` | string | `real` (actual weight) or `cubed` (dimensional weight) — which one was used in the rating |
| `carriers[].weight_details.weight` | string | Weight considered |
| `carriers[].freight` / `freight_final` | string | Freight before / after the administrative fees |
| `carriers[].insurance` / `insurance_final` | string | Insurance before / after the administrative fees |
| `carriers[].tax` / `tax_final` | string | Taxes before / after the administrative fees |
| `carriers[].currency_quote_price` | string | Total in the quotation currency |
| `carriers[].currency_quote_amount` | string | Total payable in the quotation currency |
| `carriers[].currency_payment_amount` | string | Total payable in the payment currency |

## Endpoint: query the quotation result

- **Method and route:** `GET /v1/quotation/{quotation}/{level}`
- **When to use:** to re-query an already calculated result or to change the level of detail.

| Parameter | Location | Type | Required | Values | Description |
|---|---|---|---|---|---|
| `quotation` | path | string | **yes** | UUID | Code returned on creation |
| `level` | path | string | no | `simple`, `detailed` | Default `simple` |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

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

> If the `quotation` does not exist or does not belong to your account, the API responds `400` with
> `Não foi encontrada a cotação especificada!`.

## Business rules

### Eligibility for a shipment
For a quotation to be able to generate a shipment, all of the conditions below must be true:

1. The `quotation` exists and belongs to your account.
2. The quotation **is not expired** (respect the `valid_until` of the chosen carrier).
3. The quotation **has not been used yet** in another shipment (`Cotação já foi utilizada em outra remessa!`).
4. The quotation was created in the **same version** of the API used for the shipment (v1 → v1).
5. It is not a `simple` quotation of `package`/`pallet` (`Cotação Simples é inelegível para remessa!`).
6. The informed carrier is in the list returned by the quotation
   (`Contrato especificado não foi listado para esta cotação!`).

### Cargo restrictions
The quotation can be refused with `400` when:

| Message | Cause |
|---|---|
| `A caixa :name tem a altura/largura/profundidade maior do que a permitida!` | Dimension above the contracts' limit |
| `A caixa :name está com o peso maior que o permitido!` | Weight per package above the limit |
| `Remessa está com o peso acima do permitido!` | Total weight above the limit |
| `Nenhum item foi associado para a caixa :name !` | Box without an item in `advanced` |
| `A cotação excedeu o máximo de itens permitido!` | Number of items above the limit |
| `Nenhum transportador encontrado para este usuário!` | Account without an enabled contract |
| `Não há contratos que atendem as especificações do pacote enviado...` | No contract covers the weight/dimension/route combination |
| `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Cargo outside the automatic range — talk to the sales team |
| `Endereço de Remetente e Destinatário são iguais!` | Identical origin and destination |
| `O país de origem é o mesmo de destino!` | The API serves international shipments |
| `Não foi passado um postal code para o endereço do Brasil no :name !` | Brazilian address without a postal code |
| `Este país de Origem/Destino está temporariamente inabilitado para envios!` | Route currently blocked |
| `Não existe tabela de duty para o país de destino :name com o hscode :hscode...` | HS Code without a tax table at the destination — review the `hscode` |

### Referenced registrations
When you use registration codes, they must exist and be active:

| Field | Registration | Page |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Box | [Box](api-caixa-v1.md) |
| `items[].sku` | Item | [Items](api-itens-v1.md) |
| `address_*.address_code` | Address | Registered in the panel |
| `channel_code` | Sales channel | Registered in the panel |
| `boxes[].kit` | Kit | Registered in the panel |

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Business rule (cargo restriction, route, contract, HS Code) | Read `message` and adjust the scenario |
| `403` | No quotation permission | Ask support for the permission |
| `406` | Incompatible payload structure | Review types and nesting |
| `422` | Field validation | Fix the fields listed in `data` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- The `POST` response already brings the calculated result — it is not mandatory to call the `GET` afterwards.
- `data.warnings` is informative: the quotation was performed, but there is a customs risk to review.
- For an even leaner quotation payload (only country and state), see [Quotation V2](api-cotacao-v2.md).
- For quotation + shipment in a single synchronous call, see [Manifest](api-manifesto.md).
