# Shipment (v2)

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md) and [Quotation (v2)](api-cotacao-v2.md)

## Overview
- Turns a **v2** quotation into a real shipment.
- The central difference compared to [Shipment v1](api-remessa-v1.md): here you inform the **full address**
  (city, state and postal code), because the v2 quotation only received the country.
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
1. POST /v2/quotation                 → quotation + carriers[]   (country/state only)
2. POST /v2/shipping                  → shipment received         (full address)
3. GET  /v1/shipping/{shipping}/status→ follow until "shipped"
4. GET  /v1/shipping/{shipping}/docs  → download the AWB, invoice, etc.
5. POST /v1/schedule                  → schedule pickup (optional)
```

1. [Create the v2 quotation](api-cotacao-v2.md) and check whether it is eligible for a shipment.
2. Send the shipment with the `quotation`, the chosen carrier and the full address.
3. Creation is **asynchronous**: `200` means "we received it", not "we dispatched it".
4. Follow it through the [tracking](api-rastreio.md) and download the [documents](api-documentos.md).

## Endpoint map by version

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

> There is no `GET /v2/shipping/...` or `DELETE /v2/shipping/...`. Use the `/v1` routes.

## Endpoint: create shipment

- **Method and route:** `POST /v2/shipping`
- **When to use:** after getting an eligible **v2** quotation, within the validity period.

### Parameters — payload root

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `quotation` | string | **yes** | - | - | - | Quotation UUID. It must have been created in **v2** |
| `carrier_code` | integer | **yes** if `slug` is not sent | - | - | - | Contract code (`carriers[].code` from the quotation) |
| `slug` | string | **yes** if `carrier_code` is not sent | - | 20 | - | Text identifier of the contract (`carriers[].slug` from the quotation) |
| `address_sender` | object | **yes** | - | - | - | Sender (see table below) |
| `address_receiver` | object | **yes** | - | - | - | Receiver (see table below) |
| `payment` | object | **yes** | - | - | - | Payment method |
| `type` | string | no | - | 30 | `Commercial` | Purpose of the shipment. Values: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | no | 44 | 44 | `null` | NF-e key. **Exactly 44 digits** |
| `invoice_number` | string | no | 1 | 50 | internal code | Invoice number |
| `shipment_reference` | string | no | 1 | 50 | internal ID | Shipment reference |
| `cost_center_code` | integer | no | - | - | `null` | Cost center code. It must exist and be released for your user |
| `payment_notify_shipping_billing` | boolean | no | - | - | `false` | Notify the billing area about the charge |
| `attachments` | array | no | - | - | `null` | Documents attached to the shipment |
| `biller` | object | no | - | - | account preset | Biller shown on the invoice instead of the sender |

### Parameters — `address_sender` and `address_receiver`

**This is the main difference in v2:** city, state and postal code are required here.

**Form A — address already registered:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this form | Code of the registered address. When used, do not send the other fields |

**Form B — manual address:**

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `city` | string | **yes** | 3 | 100 | - | City |
| `postal_code` | string | conditional | - | 12 | - | Postal/ZIP code. Required if `state` is not informed. **Always required for addresses in Brazil** |
| `state` | string | conditional | - | - | - | Name of the state/province. Required if `postal_code` is not informed |
| `state_code` | string | conditional | - | - | - | State/province abbreviation. Required if `state` and `postal_code` are not informed |
| `street` | string | **yes** | - | 70 | - | Street |
| `name` | string | **yes** | 5 | 35 | - | Contact name/legal name |
| `email` | string | **yes** | - | 60 | - | Valid e-mail (receives the carrier's notifications) |
| `phone` | string | **yes** | - | 25 | - | Phone |
| `type` | string | **yes** | - | - | `pj` | `pf` (individual) or `pj` (company) |
| `number` | string | no | - | 6 | `null` | Building number |
| `district` | string | no | - | 30 | `null` | District |
| `complement` | string | no | - | 20 | `null` | Complement |
| `description` | string | no | - | 155 | `null` | Internal label of the address |
| `phone_extension` | string | no | - | 5 | `null` | Extension |
| `federal_tax_id` | string | conditional | - | 20 | `null` | CPF/CNPJ or federal document. **Required when the country is `BR` or `AR`** |
| `state_tax_id` | string | conditional | - | 20 | `null` | State tax registration. **Required when the country is `BR`** |
| `foreign` | boolean | conditional | - | - | `false` | Indicates whether the person is a foreigner. **Required when the country is `BR` and `address_code` is not used** |

> **Rule of thumb:** inform `city` + (`postal_code` **or** `state_code`) + `street` + `name` + `email` +
> `phone` + `type`. The country comes from the quotation.

### Parameters — `payment`

| Field | Type | Required | Values | Description |
|---|---|---|---|---|
| `payment.type` | string | **yes** | `billed`, `gateway`, `afterward` | Financial model of the shipment |

| Value | Meaning |
|---|---|
| `billed` | Billed: the amount goes into your periodic invoice |
| `gateway` | Immediate payment via gateway, in the currency defined in `currency_payment` |
| `afterward` | Later payment, settled outside the automatic flow |

### Parameters — `attachments[]`

| Field | Type | Required | Max. | Description |
|---|---|---|---|---|
| `description` | string | **yes** | 100 | Name/description of the document (used later to locate or remove the attachment) |
| `base64` | string | **yes** | - | File content encoded in Base64 |

| Limit | Value |
|---|---|
| Accepted formats | `PDF`, `XML`, `JPEG`, `PNG` |
| Size per file | **5 MB** |

### Parameters — `biller` (invoicing party)

When informed, this address appears on the invoice instead of the sender's address.
Send `address_code` **or** the manual fields. **If you are not going to use a biller, do not send the field.**

| Field | Type | Min. | Max. | Description |
|---|---|---|---|---|
| `address_code` | integer | - | - | Code of a registered address |
| `country_code` | string | 2 | 2 | Country in ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | State name |
| `state_code` | string | - | 10 | State abbreviation |
| `postal_code` | string | - | 12 | Postal/ZIP code |
| `city` | string | 3 | 100 | City |
| `street` | string | - | 70 | Street |
| `number` | string | - | 6 | Number |
| `district` | string | - | 30 | District |
| `complement` | string | - | 20 | Complement |
| `description` | string | - | 155 | Internal label |
| `name` | string | 5 | 35 | Name/legal name |
| `email` | string | - | 60 | E-mail |
| `phone` | string | - | 25 | Phone |
| `phone_extension` | string | - | 5 | Extension |
| `type` | string | - | - | `pf` or `pj` |
| `federal_tax_id` | string | - | 20 | Federal document |
| `state_tax_id` | string | - | 20 | State tax registration |
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
    "phone": "11992494597",
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
    "type": "gateway"
  },
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcKJcTl8uXrp..."
    },
    {
      "description": "Nota Fiscal XML",
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
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "state_code": "FL", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

## Differences from v1

| Topic | v1 | v2 |
|---|---|---|
| Address in the shipment | Street, name, e‑mail, phone, type (+ tax documents) | The same fields **+ `city`, `state`/`state_code`, `postal_code`** |
| `city` | Not accepted (comes from the quotation) | **Required**, min. 3 and max. 100 characters |
| `biller` | Locality fields accepted | Locality fields **validated** (`city` min. 3) |
| Endpoints of the cycle | Creation, status, documents, update, attachments, cancellation | **Creation only.** The rest uses `/v1` |
| Origin of the quotation | `POST /v1/quotation` | `POST /v2/quotation` |

## Business rules

### Quotation eligibility
| Condition | Message when it fails |
|---|---|
| The quotation exists and belongs to the account | `Nenhuma cotação foi encontrada com os dados especificados!` |
| The quotation is not expired | `Esta cotação está expirada...` |
| The quotation has not been used before | `Cotação já foi utilizada em outra remessa!` |
| Created in **v2** | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| It is not a `simple` quotation of `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| The contract is in the quotation's list | `Contrato especificado não foi listado para esta cotação!` |

### Consistency between quotation and shipment
The country informed in the shipment must be the same as in the quotation:
- `O país :country é diferente do enviado na cotação!`
- `O Estado :state é diferente do enviado na cotação!`
- `A cidade :city é diferente do enviado na cotação!`

### Required tax documents
| Address country | `federal_tax_id` | `state_tax_id` | `foreign` |
|---|---|---|---|
| `BR` | Required (valid CPF or CNPJ) | Required | Required without `address_code` |
| `AR` | Required | - | - |
| Others | Optional | Optional | Optional |

- For `type: pf`, the state tax registration cannot be `ISENTO`/`ISENTA`.

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Business rule (quotation, address, payment, tax document) | Read `message` and fix the scenario |
| `401` | Payment not authorized | Enable the payment type in the account |
| `403` | No permission for the action | Ask support for the permission |
| `404` | `Contrato especificado não foi listado para esta cotação!` | Check `carrier_code`/`slug` |
| `422` | Field validation or quotation from an incompatible version | Fix the fields listed in `data` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- Use v2 when the full address only exists at shipping time.
- The entire post-creation operational cycle is handled by the `/v1` endpoints — they do not distinguish the
  version in which the shipment was created.
- For the details of each post-creation operation, see [Shipment (v1)](api-remessa-v1.md),
  [Tracking](api-rastreio.md), [Documents](api-documentos.md) and [Scheduling](api-agendamento.md).
