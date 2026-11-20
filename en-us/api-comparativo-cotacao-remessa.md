# Quotation & Shipment: v1 vs v2

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
There are two published versions of the **quotation + shipment** pair. They use the **same calculation**
and the **same carriers**. The differences lie in **when** you provide the full address and in
**how** you describe the cargo: only v2 accepts the quotation by items, without providing the boxes.

| | v1 | v2 |
|---|---|---|
| Full address | Provided **in the quotation** | Provided **in the shipment** |
| Quotation payload | Country + city + ZIP code/state | Country only (+ optional state) |
| Quotation by items (`type: items`) | Not available: boxes always provided | **Available**: only the `sku` of the registered items |
| Shipment payload | Street, name, contact | Street, name, contact **+ city, state, ZIP code** |
| Address validation | In the quotation | In the shipment |
| Full operational cycle | Yes, in `/v1` | Creation in `/v2`, the rest in `/v1` |

## In one sentence

> **Use v1** if, at quotation time, you already know which city/ZIP code you will ship to.
> **Use v2** if, at quotation time, you only know the destination country **or** want to quote only by the registered
> SKUs, without providing the boxes.

## Golden rule: do not mix versions

A quotation created in v1 **only** generates a shipment in `POST /v1/shipping`.
A quotation created in v2 **only** generates a shipment in `POST /v2/shipping`.

Mixing them returns `422`, with the message below in `data.quotation`:
```
Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.
```

## Quotation comparison

| Topic | v1 (`POST /v1/quotation`) | v2 (`POST /v2/quotation`) |
|---|---|---|
| `address_*.country_code` | Required | Required |
| `address_*.city` | **Required** | Ignored |
| `address_*.postal_code` | Required (or `state`); **always required in Brazil** | Ignored |
| `address_*.state` | Required (or `state_code`/`postal_code`) | Ignored |
| `address_*.state_code` | Required (or `state`/`postal_code`) | **Optional** (improves accuracy) |
| `object` | `doc`, `package`, `pallet`, `not_doc` | Identical |
| `type` | `simple`, `advanced` | `simple`, `advanced` **and `items`** (exclusive to v2) |
| `boxes[]` (incl. `kit`, `volume`) | **Always required** | Required, except in `type: items` (depending on the account grouping) |
| `items[]` (incl. `ean`, `nsmp`) | Supported | Identical, **without** `selling_unit_value` |
| `ior` (Importer of Record) | Supported | **Not part of the contract** |
| `declared_freight` | Supported | Identical |
| `manufacturer` | Supported | Identical |
| `level` (`simple`/`detailed`) | Supported | Identical |
| Address validation (ZIP code, city, state) | In the quotation | None (done in the shipment) |
| Retrieve the result later | `GET /v1/quotation/{quotation}/{level}` | **Does not exist** |
| Customs warnings | Supported | Identical |
| Account presets (tax, automatic insurance, etc.) | Applied | Identical |
| Scenarios (document, package, pallet, items, kit) | [v1 scenarios](api-cotacao-v1.md#quotation-scenarios) | [v2 scenarios](api-cotacao-v2.md#quotation-scenarios) |

### Practical impact on the quotation
- **Accuracy:** in v1, city and ZIP code allow regional fees and restrictions to be applied **already in the calculation**.
  In v2, the calculation considers country and state; the full address only comes in with the shipment.
- **Address errors:** in v1 they show up in the quotation; in v2 they show up in the shipment.
- **Re-query:** v2 has no re-query endpoint. **Store the `POST` response** if you need to
  display the quotation to the user again.

## Shipment comparison

| Topic | v1 (`POST /v1/shipping`) | v2 (`POST /v2/shipping`) |
|---|---|---|
| `quotation` | v1 quotation | v2 quotation |
| `carrier_code` / `slug` | One of the two | Identical |
| `address_*.street`, `name`, `email`, `phone`, `type` | Required | Identical |
| `address_*.city` | Ignored (comes from the quotation) | **Required** (min. 3, max. 100) |
| `address_*.postal_code` | Ignored | **Required** if `state` is not sent |
| `address_*.state` | Ignored | **Required** if `postal_code` is not sent |
| `address_*.state_code` | Ignored | Optional; the one from the quotation prevails |
| `address_*.address_code` | Must match the quotation address | Only works if the v2 quotation used the same `address_code` |
| `payment.type` | `billed`, `gateway`, `afterward` | Identical |
| `nfkey`, `invoice_number`, `shipment_reference` | Supported | Identical |
| `cost_center_code`, `payment_notify_shipping_billing` | Supported | Identical |
| `attachments` | Supported | Identical |
| `biller` | Supported | Identical, with validated location |
| Quotation eligibility | [Ineligible v1](api-remessa-v1.md#quotations-ineligible-for-shipment) | [Ineligible v2](api-remessa-v2.md#quotations-ineligible-for-shipment) |

## Post-creation operations

All operations after the shipment is created use **the same `/v1` endpoints**, regardless of the
version in which the shipment was created.

| Operation | Route | Works for v1 shipment? | Works for v2 shipment? |
|---|---|---|---|
| Query status | `GET /v1/shipping/{shipping}/status` | Yes | **Yes** |
| Status by channel | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` | Yes | **Yes** |
| Query documents | `GET /v1/shipping/{shipping}/docs` | Yes | **Yes** |
| Documents by channel | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` | Yes | **Yes** |
| Update (NF-e, attachments, cost center) | `PUT /v1/shipping/{shipping}` | Yes | **Yes** |
| Remove attachments | `DELETE /v1/shipping/{shipping}/docs` | Yes | **Yes** |
| Cancel | `DELETE /v1/shipping/{shipping}` | Yes | **Yes** |
| Cancel by channel | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` | Yes | **Yes** |
| Schedule pickup | `POST /v1/schedule` | Yes | **Yes** |
| Query scheduling | `GET /v1/schedule/{shipping}` | Yes | **Yes** |
| Cancel scheduling | `DELETE /v1/schedule/{shipping}/{reason}` | Yes | **Yes** |

> There are no `GET`, `PUT` or `DELETE` routes under `/v2/shipping`.

## Side-by-side payload comparison

### Quotation

**v1 — full address in the quotation:**
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
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "Cotton t-shirt", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

**v2 — only country/state in the quotation:**
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
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "Cotton t-shirt", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

### Shipment

**v1 — without location (it came from the quotation):**
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Rua Principal",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
    "type": "pf"
  },
  "payment": { "type": "billed" }
}
```

**v2 — with location (it did not come from the quotation):**
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
    "type": "pf"
  },
  "payment": { "type": "billed" }
}
```

## How to choose

Answer the questions in order:

1. **At quotation time, do you already know the destination city and ZIP code?**
   - Yes → **v1**
   - No → **v2**
2. **Do you need to re-query the quotation later, in another request?**
   - Yes → **v1** (`GET /v1/quotation/{quotation}/{level}`)
   - No → either one
3. **Do you use `ior` (Importer of Record) or `selling_unit_value`?**
   - Yes → **v1**
   - No → either one
4. **Do you want the highest level of tax accuracy already in the quotation?**
   - Yes → **v1**
   - No → either one
5. **Do you want to quote only with the registered SKUs, without providing the boxes?**
   - Yes → **v2** ([Items Quotation](api-cotacao-v2.md#items-quotation))
   - No → either one

### Typical scenarios

| Scenario | Recommended version | Reason |
|---|---|---|
| E-commerce checkout (address already filled in) | **v1** | Accurate quotation, shown to the end customer |
| Freight calculator on the product page (country only) | **v2** | The customer has not provided the address yet |
| Marketplace that receives the full order via webhook | **v1** | All the data already exists |
| B2B integration with batch dispatch | **v1** | Full operational flow in the same version |
| Simulation on an international landing page | **v2** | Minimum payload, less friction |
| ERP with an already registered product catalog | **v2** | Quotation by items: only SKU and quantity, the platform assembles the boxes |
| Quotation + shipment + documents in a single call | **[Manifest](api-manifesto.md)** | Synchronous flow, independent of v1/v2 |

## Other ways to create a shipment

Besides the quotation + shipment pair, there are two alternative flows:

| Flow | Route | Characteristic |
|---|---|---|
| [Manifest](api-manifesto.md) | `POST /v1/manifest` | **Synchronous**: quotes, creates the shipment with the carrier and returns the documents in the same response. Only accepts `billed` payment |
| [Draft](api-rascunho.md) | `POST /v1/draft` | **Asynchronous**: accepts a partial payload, validates it later and evolves into quotation + shipment. Accepts `billed` or `afterward` payment |

## Final notes
- The two versions coexist: you can use v2 in the storefront and v1 in the checkout, as long as **each quotation
  generates the shipment in its own version**.
- The support endpoints ([Box](api-caixa-v1.md), [Items](api-itens-v1.md),
  [Contract](api-contrato-v1.md)) are shared and exist only in `/v1`.
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
