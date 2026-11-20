# Shipment (v2)

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md) and [Quotation (v2)](api-cotacao-v2.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Turns a **v2 quotation** into a real shipment.
- The central difference compared to [Shipment v1](api-remessa-v1.md): here you provide the **full address**
  (city, state and postal code), because the v2 quotation only received the country (and perhaps the state).
- The full address is **validated in the shipment** (CEP in Brazil, international address in the other countries).
- v2 exposes **only the creation**. Status, documents, cancellation, update and pickup scheduling
  use the `/v1/...` endpoints, which work normally for shipments created in v2.

> ⚠️ **The shipment is an irreversible operation in the normal route.** Once created, it goes into processing
> at the carrier.

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

## Usage flow

```
1. POST /v2/quotation                  → quotation + carriers[]   (country/state only)
2. POST /v2/shipping                   → shipment received         (full address)
3. GET  /v1/shipping/{shipping}/status → follow until "shipped"
4. GET  /v1/shipping/{shipping}/docs   → download the AWB, invoice etc.
5. POST /v1/schedule                   → schedule pickup (optional)
```

1. [Create the v2 quotation](api-cotacao-v2.md) and check whether it is [eligible](#quotations-ineligible-for-shipment).
2. Send the shipment with the `quotation`, the chosen carrier and the full address.
3. Creation is **asynchronous**: `200` means "we received it", not "we dispatched it".
4. Follow it through the [tracking](api-rastreio.md) and download the [documents](api-documentos.md).

## Key concepts

### Shipment from a v2 quotation
- The origin and destination **country** always comes from the quotation (it is not provided in the shipment).
- The **state** provided in the quotation (`state_code`) is reused; if the quotation had no state, it is
  discovered from the postal code/address sent in the shipment.
- **City and postal code** are provided here and become valid for the shipment.
- A quotation generates **a single** shipment.

### Endpoint map by version

| Operation | Route |
|---|---|
| Create shipment | `POST /v2/shipping` |
| Query status | `GET /v1/shipping/{shipping}/status` |
| Query status by channel | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` |
| Query documents | `GET /v1/shipping/{shipping}/docs` |
| Query documents by channel | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` |
| Update (NF-e, attachments, cost center) | `PUT /v1/shipping/{shipping}` |
| Remove attachments | `DELETE /v1/shipping/{shipping}/docs` |
| Cancel | `DELETE /v1/shipping/{shipping}` |
| Cancel by channel | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` |
| Schedule pickup | `POST /v1/schedule` |

> There is no `GET /v2/shipping/...` or `DELETE /v2/shipping/...`. Use the `/v1` routes, documented in
> [Shipment (v1)](api-remessa-v1.md).

## Endpoint: create shipment

- **Method and route:** `POST /v2/shipping`
- **When to use:** after getting an eligible **v2** quotation, within the validity period.

### Parameters — payload root

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `quotation` | string | **yes** | - | - | - | Quotation UUID. It must have been created in **v2** |
| `carrier_code` | integer | **yes** if `slug` is not sent | - | - | - | Contract code (`carriers[].code` from the quotation) |
| `slug` | string | **yes** if `carrier_code` is not sent | - | 20 | - | Text identifier of the contract (`carriers[].slug` from the quotation). When sent, it takes precedence over `carrier_code` |
| `address_sender` | object | **yes** | - | - | - | Sender (see table below) |
| `address_receiver` | object | **yes** | - | - | - | Receiver (see table below) |
| `payment` | object | **yes** | - | - | - | Payment method |
| `type` | string | no | - | - | account preset or `Commercial` | Purpose of the shipment. Values: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | conditional | 44 | 44 | `null` | NF-e key. **Exactly 44 characters**. Required in some contracts |
| `invoice_number` | string | no | 1 | 50 | internal code | Invoice number |
| `shipment_reference` | string | no | 1 | 50 | internal ID | Shipment reference |
| `cost_center_code` | integer | no | - | - | account preset | Code of the cost center authorized for your user. Empty/null receives the account default |
| `payment_notify_shipping_billing` | boolean | no | - | - | account preset | With `payment.type: afterward`, sends the billing e-mail |
| `attachments` | array | no | - | - | `null` | Documents attached to the shipment |
| `biller` | object | no | - | - | account preset | Biller displayed on the invoice instead of the sender |

### Parameters — `address_sender` and `address_receiver`

**This is the main difference of v2:** city and postal code (or state) are required here.

**Form A — already registered address:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this form | Code of the registered address. When used, do not send the other fields |

> ⚠️ In v2, use `address_code` in the shipment **only if the same `address_code` was used in the v2 quotation**.
> A quotation made with the country only has no city, and the registered address will be refused with
> `406 A cidade :city é diferente do enviado na cotação!`.

**Form B — manual address:**

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `city` | string | **yes** | 3 | 100 | - | City |
| `postal_code` | string | conditional | - | 12 | - | Postal code/ZIP. Required if `state` is not provided. **Always recommended; in Brazil it is what allows the address to be validated** |
| `state` | string | conditional | - | - | - | State/province name. Required if `postal_code` is not provided |
| `state_code` | string | no | - | - | quotation state | State/province abbreviation. When the quotation had a `state_code`, that one is used instead of this |
| `street` | string | **yes** | - | 70 | - | Street |
| `name` | string | **yes** | 5 | 35 | - | Contact name/company name |
| `email` | string | **yes** | - | 60 | - | Valid e-mail (receives carrier notifications) |
| `phone` | string | **yes** | - | 25 | - | Phone |
| `type` | string | **yes** | - | - | - | `pf` (individual) or `pj` (legal entity) |
| `number` | string | no | - | 6 | `null` | Building number |
| `district` | string | no | - | 30 | `null` | District |
| `complement` | string | no | - | 20 | `null` | Address complement |
| `description` | string | no | - | 155 | `null` | Internal label of the address |
| `phone_extension` | string | no | - | 5 | `null` | Phone extension |
| `federal_tax_id` | string | conditional | - | 20 | `null` | CPF/CNPJ or federal document. Required depending on the country (see [Tax documents](api-remessa-v1.md#required-tax-documents)) |
| `state_tax_id` | string | conditional | - | 20 | `null` | State registration. Required for Brazilians (`BR` with `foreign: false`) |
| `foreign` | boolean | no | - | - | `false` if the country is `BR`; `true` otherwise | Indicates whether the person is a foreigner |

> **Practical rule:** provide `city` + `postal_code` + `street` + `name` + `email` + `phone` + `type`
> (+ tax documents when required). The country comes from the quotation.
> Without `postal_code`, the `state` field (state name) becomes required — `state_code` alone is not enough.

### Parameters — `payment`

| Field | Type | Required | Values | Description |
|---|---|---|---|---|
| `payment.type` | string | **yes** | `billed`, `gateway`, `afterward` | Financial model of the shipment |

| Value | Meaning | Account requirement |
|---|---|---|
| `billed` | Billed: the amount goes into your periodic invoice | "Billed" payment option enabled |
| `gateway` | Immediate payment via gateway, in the currency defined in `currency_payment` | "Immediate" payment option enabled |
| `afterward` | Later payment, settled outside the automatic flow | - |

### Parameters — `attachments[]`

| Field | Type | Required | Max. | Description |
|---|---|---|---|---|
| `description` | string | **yes** | 100 | Name/description of the document (used later to find or remove the attachment) |
| `base64` | string | **yes** | - | File content encoded in Base64 |

| Limit | Value |
|---|---|
| Accepted formats | `PDF`, `XML`, `JPEG`, `PNG` |
| Total request size | **90 MB** |

> As in v1, attachments in an unrecognized format are **silently discarded** on creation. To attach
> with validation, use `PUT /v1/shipping/{shipping}` (limit of 5 MB per file).

### Parameters — `biller`

When provided, this address appears on the invoice instead of the sender's address.
Send `address_code` **or** the manual fields. **If you are not going to use a biller, do not send the field.**

| Field | Type | Min. | Max. | Description |
|---|---|---|---|---|
| `address_code` | integer | - | - | Registered address code |
| `country_code` | string | 2 | 2 | Country in ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | State name |
| `state_code` | string | - | 10 | State abbreviation |
| `postal_code` | string | - | 12 | Postal code/ZIP |
| `city` | string | 3 | 100 | City |
| `street` | string | - | 70 | Street |
| `number` | string | - | 6 | Number |
| `district` | string | - | 30 | District |
| `complement` | string | - | 20 | Address complement |
| `description` | string | - | 155 | Internal label |
| `name` | string | 5 | 35 | Name/company name |
| `email` | string | - | 60 | E-mail |
| `phone` | string | - | 25 | Phone |
| `phone_extension` | string | - | 5 | Phone extension |
| `type` | string | - | - | `pf` or `pj` |
| `federal_tax_id` | string | - | 20 | Federal document |
| `state_tax_id` | string | - | 20 | State registration |
| `foreign` | boolean | - | - | Indicates whether it is a foreigner |

### Minimum payload

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
    "state_tax_id": "123456789012"
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
  "payment": {
    "type": "billed"
  }
}
```

### Maximum payload

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "slug": "dhl-br",
  "type": "Commercial",
  "nfkey": "12345678901234567890123456789012345678901234",
  "invoice_number": "NF-000123",
  "shipment_reference": "REF-2026-0001",
  "cost_center_code": 1,
  "payment_notify_shipping_billing": true,
  "address_sender": {
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
    "phone": "13055550100",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Headquarters",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Room 1",
    "name": "Holding Exemplo LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "phone_extension": "10",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "payment": {
    "type": "afterward"
  },
  "attachments": [
    {
      "description": "Invoice",
      "base64": "JVBERi0xLjcKJcTl8uXrp..."
    },
    {
      "description": "Invoice XML",
      "base64": "PD94bWwgdmVyc2lvbj0i..."
    }
  ]
}
```

### Request example

```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success` when the shipment was registered |
| `message` | string | Receipt confirmation. The shipment identifier is the `quotation` itself |

## Differences compared to v1

| Topic | v1 | v2 |
|---|---|---|
| Address in the shipment | Street, name, e‑mail, phone, type (+ tax documents) | The same fields **+ `city`, `postal_code` (or `state`)** |
| `city` | Ignored (comes from the quotation) | **Required**, min. 3 and max. 100 characters |
| Address validation | Done in the quotation | **Done in the shipment** |
| `address_code` | Must match the quotation address | Only works if the v2 quotation also used `address_code` |
| `biller` | Location fields accepted | Location fields **validated** (`city` min. 3) |
| Lifecycle endpoints | Creation, status, documents, update, attachments, cancellation | **Creation only.** The rest uses `/v1` |
| Quotation origin | `POST /v1/quotation` | `POST /v2/quotation` |

## Quotations ineligible for shipment

The same checks as in v1 apply, in the same order — see the full table in
[Shipment (v1)](api-remessa-v1.md#quotations-ineligible-for-shipment). The only difference is the version:

| Check | Status | Message |
|---|---|---|
| `quotation` exists and was created in **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| Simple package/pallet quotation | `403` | `Cotação Simples é inelegível para remessa!` |
| Quotation already used | `406` | `Cotação já foi utilizada em outra remessa!` |
| Expired quotation | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| Contract outside the quotation list | `404` | `Contrato especificado não foi listado para esta cotação!` |

## Business rules

### Full address validation
After the quotation checks, the manual address (Form B) is validated:

| Case | Status | Message |
|---|---|---|
| Brazil, without state in the quotation: nonexistent CEP or CEP from another state | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| Outside Brazil: address not found | `404` | `Endereço internacional enviado está errado ou não existe!` |
| Outside Brazil: state/province not recognized | `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |

> Always send `postal_code` and, when you know it, `state_code` (preferably already in the quotation): this gives the highest
> success rate in the validation.

### Consistency between quotation and shipment
- The country is always the one from the quotation.
- The quotation's `state_code`, when it exists, takes precedence over the one sent in the shipment.
- With `address_code`, the registered address must have the same country, state and city recorded in the quotation:
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Tax documents, carrier and presets
Identical to v1:
[Required tax documents](api-remessa-v1.md#required-tax-documents),
[Carrier restrictions](api-remessa-v1.md#carrier-restrictions) and
[Presets applied to the shipment](api-remessa-v1.md#presets-applied-to-the-shipment).

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Tax document, invalid Brazilian CEP, required NF-e | Read `message` and fix the scenario |
| `401` | Payment not authorized | Enable the payment type in the account |
| `403` | Simple/expired quotation, or no permission | Redo the quotation / request the permission from support |
| `404` | Contract outside the quotation, international address not found | Check `carrier_code`/`slug` and the address |
| `406` | Quotation already used or incompatible `address_code` | Make a new quotation / use a manual address |
| `422` | Field validation, quotation from another version, DDP-only account | Fix the fields listed in `data` |
| `500` | Unexpected error or state/province not recognized | Review the address; retry with backoff; if it persists, contact support |

## Final notes
- Use v2 when the full address only exists at the time of shipping.
- The entire post-creation operational cycle is done through the `/v1` endpoints — they do not distinguish the version
  in which the shipment was created.
- For details on each post-creation operation, see [Shipment (v1)](api-remessa-v1.md),
  [Tracking](api-rastreio.md), [Documents](api-documentos.md) and [Scheduling](api-agendamento.md).
- Differences between the versions in [Quotation & Shipment: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
