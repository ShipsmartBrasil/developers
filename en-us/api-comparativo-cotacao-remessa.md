# Quotation & Shipment: v1 vs v2

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
There are two published versions for the **quotation + shipment** pair. They use the **same calculation
engine** and the **same carriers**. The difference is **when** you inform the full address.

| | v1 | v2 |
|---|---|---|
| Full address | Informed **in the quotation** | Informed **in the shipment** |
| Quotation payload | Country + city + postal code/state | Only the country (+ optional state) |
| Shipment payload | Street, name, contact | Street, name, contact **+ city, state, postal code** |
| Full operational cycle | Yes, under `/v1` | Creation under `/v2`, the rest under `/v1` |

## In one sentence

> **Use v1** if, at quotation time, you already know which city/postal code you will ship to.
> **Use v2** if, at quotation time, you only know the destination country.

## Golden rule: do not mix versions

A quotation created in v1 **only** generates a shipment in `POST /v1/shipping`.
A quotation created in v2 **only** generates a shipment in `POST /v2/shipping`.

Mixing them returns `422`:
```
Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.
```

## Quotation comparison

| Topic | v1 (`POST /v1/quotation`) | v2 (`POST /v2/quotation`) |
|---|---|---|
| `address_*.country_code` | Required | Required |
| `address_*.city` | **Required** | Not accepted |
| `address_*.postal_code` | Required (or `state`) | Not accepted |
| `address_*.state` | Required (or `state_code`/`postal_code`) | Not accepted |
| `address_*.state_code` | Required (or `state`/`postal_code`) | **Optional** (improves accuracy) |
| `object` | `doc`, `package`, `pallet`, `not_doc` | Identical |
| `type` | `simple`, `advanced`, `items` | Identical |
| `boxes[]` (incl. `kit`, `volume`) | Supported | Identical |
| `items[]` | Supported | Identical, **without** `selling_unit_value` |
| `ior` (Importer of Record) | Supported | **Not part of the contract** |
| `declared_freight` | Supported | Identical |
| `manufacturer` | Supported | Identical |
| `level` (`simple`/`detailed`) | Supported | Identical |
| Query the result later | `GET /v1/quotation/{quotation}/{level}` | **Does not exist** |
| Customs warnings | Supported | Identical |

### Practical impact on the quotation
- **Accuracy:** in v1, city and postal code allow regional fees and restrictions to be applied **already in
  the calculation**. In v2, the amount may be refined when the full address is entered in the shipment.
- **Re-query:** v2 has no re-query endpoint. **Store the `POST` response** if you need to
  show the quotation to the user again.

## Shipment comparison

| Topic | v1 (`POST /v1/shipping`) | v2 (`POST /v2/shipping`) |
|---|---|---|
| `quotation` | v1 quotation | v2 quotation |
| `carrier_code` / `slug` | One of the two | Identical |
| `address_*.street`, `name`, `email`, `phone`, `type` | Required | Identical |
| `address_*.city` | Not accepted | **Required** (min. 3, max. 100) |
| `address_*.postal_code` | Not accepted | **Required** (or `state`) |
| `address_*.state` / `state_code` | Not accepted | **Required** (one of them, or `postal_code`) |
| `payment.type` | `billed`, `gateway`, `afterward` | Identical |
| `nfkey`, `invoice_number`, `shipment_reference` | Supported | Identical |
| `cost_center_code`, `payment_notify_shipping_billing` | Supported | Identical |
| `attachments` | Supported | Identical |
| `biller` | Supported | Identical, with a validated locality |

## Post-creation operations

All operations after the shipment is created use **the same `/v1` endpoints**, regardless of the
version in which the shipment was created.

| Operation | Route | Works for a v1 shipment? | Works for a v2 shipment? |
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

**v1 — without locality (it came from the quotation):**
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
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": { "type": "billed" }
}
```

**v2 — with locality (it did not come from the quotation):**
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
    "state_code": "FL",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": { "type": "billed" }
}
```

## How to choose

Answer the questions in order:

1. **At quotation time, do you already know the destination city and postal code?**
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

### Typical scenarios

| Scenario | Recommended version | Reason |
|---|---|---|
| E-commerce checkout (address already filled in) | **v1** | Accurate quotation, shown to the end customer |
| Freight calculator on the product page (country only) | **v2** | The customer has not informed the address yet |
| Marketplace that receives the complete order via webhook | **v1** | All data already exists |
| B2B integration with batch dispatch | **v1** | Full operational flow in the same version |
| Simulation on an international landing page | **v2** | Minimum payload, less friction |
| Quotation + shipment + documents in one call | **[Manifest](api-manifesto.md)** | Synchronous flow, independent of v1/v2 |

## Other ways to create a shipment

Besides the quotation + shipment pair, there are two alternative flows:

| Flow | Route | Characteristic |
|---|---|---|
| [Manifest](api-manifesto.md) | `POST /v1/manifest` | **Synchronous**: quotes, creates the shipment at the carrier and returns the documents in the same response. Only accepts `billed` payment |
| [Draft](api-rascunho.md) | `POST /v1/draft` | **Asynchronous**: accepts a partial payload, validates it later and evolves into quotation + shipment. Accepts `billed` or `afterward` payment |

## Final notes
- The two versions coexist: you can use v2 on the storefront and v1 at checkout, as long as **each quotation
  generates the shipment in its own version**.
- The support endpoints ([Box](api-caixa-v1.md), [Items](api-itens-v1.md),
  [Contract](api-contrato-v1.md)) are shared and exist only under `/v1`.
