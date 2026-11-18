# Shipment (v1)

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md) and [Quotation (v1)](api-cotacao-v1.md)

## Overview
- Turns a v1 quotation into a real shipment: generates the AWB, invoice, declaration and the other documents.
- It is the most complete module of the cycle: creation, status query, document query, update,
  attachment removal and cancellation.
- The status, documents and cancellation endpoints **also** serve shipments created in
  [Shipment V2](api-remessa-v2.md).

> ⚠️ **The shipment is an irreversible operation in the normal route.** Once created, it goes into processing
> at the carrier. The only later changes allowed are exceptional: linking the NF-e when it
> was not informed, attaching documents that were not sent at creation and informing the cost center.

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
1. POST /v1/quotation                 → quotation + carriers[]
2. POST /v1/shipping                  → shipment received (asynchronous)
3. GET  /v1/shipping/{shipping}/status→ follow until "shipped"
4. GET  /v1/shipping/{shipping}/docs  → download the AWB, invoice, etc.
5. POST /v1/schedule                  → schedule pickup (optional)
```

1. [Create the quotation](api-cotacao-v1.md) and check whether it is **eligible** for a shipment.
2. Send the shipment with the `quotation` and the chosen carrier (`carrier_code` **or** `slug`).
3. Creation is **asynchronous**: a `200` response means "we received it", not "we dispatched it".
4. Query the [status](api-rastreio.md) until it appears in `shipped`.
5. Download the [documents](api-documentos.md).
6. If necessary, [schedule the pickup](api-agendamento.md).

## The `{shipping}` identifier

In all query/change endpoints, the `{shipping}` path parameter accepts **three formats**:

| Format | Example | Note |
|---|---|---|
| Quotation UUID | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Recommended form |
| AWB number | `271249639538` | Works for shipments created in the **last 90 days** |
| NF-e key | `12345678901234567890123456789012345678901234` | Exactly 44 numeric digits |

To query without storing any of these values, use the endpoints **by channel**, based on the
`channel_identifier_number` you informed in the quotation.

## Endpoint: create shipment

- **Method and route:** `POST /v1/shipping`
- **When to use:** after getting an eligible v1 quotation, within the validity period.

### Parameters — payload root

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `quotation` | string | **yes** | - | - | - | Quotation UUID. It must have been created in **v1** |
| `carrier_code` | integer | **yes** if `slug` is not sent | - | - | - | Contract code (`carriers[].code` from the quotation) |
| `slug` | string | **yes** if `carrier_code` is not sent | - | 20 | - | Text identifier of the contract (`carriers[].slug` from the quotation) |
| `address_sender` | object | **yes** | - | - | - | Sender (see table below) |
| `address_receiver` | object | **yes** | - | - | - | Receiver (see table below) |
| `payment` | object | **yes** | - | - | - | Payment method (see table below) |
| `type` | string | no | - | 30 | `Commercial` | Purpose of the shipment. Values: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | no | 44 | 44 | `null` | NF-e key. When informed, it must have **exactly 44 digits** |
| `invoice_number` | string | no | 1 | 50 | internal code | Invoice number. If omitted, the shipment's internal code is used |
| `shipment_reference` | string | no | 1 | 50 | internal ID | Shipment reference. If omitted, the shipment's internal ID is used |
| `cost_center_code` | integer | no | - | - | `null` | Cost center code. It must exist and be released for your user |
| `payment_notify_shipping_billing` | boolean | no | - | - | `false` | Notify the billing area about the shipment charge |
| `attachments` | array | no | - | - | `null` | Documents attached to the shipment (see table below) |
| `biller` | object | no | - | - | account preset | Biller shown on the invoice instead of the sender (see table below) |

> **`carrier_code` or `slug`:** send **one** of the two. The informed contract must be in the
> `carriers[]` list returned by the quotation; otherwise the API responds `404` with
> `Contrato especificado não foi listado para esta cotação!`.

### Parameters — `address_sender` and `address_receiver`

Two forms, do not mix them:

**Form A — address already registered:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this form | Code of the registered address. **It must be the same address used in the quotation** |

**Form B — manual address:**

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
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
| `foreign` | boolean | conditional | - | - | `false` | Indicates whether the person is a foreigner. **Required when the country is `BR` and `address_code` is not used** — it is what allows the `federal_tax_id` to be validated correctly |

> The country, the state, the city and the postal code are **not** informed here: they come from the quotation.
> If you need to inform them in the shipment, use [Shipment V2](api-remessa-v2.md).

### Parameters — `payment`

| Field | Type | Required | Values | Description |
|---|---|---|---|---|
| `payment.type` | string | **yes** | `billed`, `gateway`, `afterward` | Financial model of the shipment |

| Value | Meaning |
|---|---|
| `billed` | Billed: the amount goes into your periodic invoice |
| `gateway` | Immediate payment via gateway, in the currency defined in `currency_payment` |
| `afterward` | Later payment, settled outside the automatic flow |

> The type must be **enabled for your account**. Otherwise:
> `400 Tipo de pagamento inexistente!` or `401 Opção de pagamento não autorizado!`.

### Parameters — `attachments[]`

| Field | Type | Required | Max. | Description |
|---|---|---|---|---|
| `description` | string | **yes** | 100 | Name/description of the document (used later to locate or remove the attachment) |
| `base64` | string | **yes** | - | File content encoded in Base64 |

| Limit | Value |
|---|---|
| Accepted formats | `PDF`, `XML`, `JPEG`, `PNG` |
| Size per file | **5 MB** |
| Total request size | **90 MB** |

> File in the wrong format: `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.`
> File above the limit: `O arquivo enviado excede o limite máximo de 5MB.`
> Corrupted XML: `O arquivo XML enviado é inválido ou está corrompido.`

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
| `city` | string | - | 100 | City |
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
  "payment": {
    "type": "billed"
  }
}
```

> `federal_tax_id`, `state_tax_id` and `foreign` appear in the minimum payload because the sender is Brazilian.
> For a sender from another country (except `AR`), these three fields can be omitted.

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
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

> The response does **not** bring the AWB or the documents: the shipment was only received. Use the
> [tracking](api-rastreio.md) to follow it and the [documents](api-documentos.md) to download the files.

### Errors specific to creation

| Status | Message | Cause / fix |
|---|---|---|
| `400` | `Tipo de pagamento inexistente!` | `payment.type` outside `billed`/`gateway`/`afterward` or not enabled |
| `400` | `Esta cotação está expirada. Porfavor, faça outra cotação...` | `valid_until` exceeded — redo the quotation |
| `400` | `Cotação já foi utilizada em outra remessa!` | One quotation generates **one** shipment |
| `400` | `Cotação Simples é inelegível para remessa!` | Use `type: advanced` or `items` in the quotation |
| `400` | `Chave da Nota não foi informada!!` | The contract requires `nfkey` for this scenario |
| `400` | `O endereço especificado, não foi encontrado!` | `address_code` invalid or from another account |
| `400` | `O país :country é diferente do enviado na cotação!` | The shipment address must match the quotation one |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Country `BR`/`AR` without `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Country `BR` without `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ with an incorrect check digit |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF with an incorrect check digit |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` with an exempt `state_tax_id` |
| `401` | `Opção de pagamento não autorizado!` | Enable the payment type in your account |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` outside the quotation's list |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | A v2 quotation being used in `/v1/shipping` |

## Endpoint: query shipment status

- **Method and route:** `GET /v1/shipping/{shipping}/status`
- **By channel:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `shipping` | path | string | **yes** | Quotation UUID, AWB (last 90 days) or NF-e key (44 digits) |
| `channel_identifier_number` | path | string | **yes** | Commercial identifier informed in the quotation |
| `channel_code` | path | integer | no | Sales channel code |

Details, response formats and status codes in [Shipment Tracking](api-rastreio.md).

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

## Endpoint: query documents

- **Method and route:** `GET /v1/shipping/{shipping}/docs`
- **By channel:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

| Parameter | Location | Type | Required | Default | Description |
|---|---|---|---|---|---|
| `shipping` | path | string | **yes** | - | UUID, AWB or NF-e key |
| `attachments` | query | boolean | no | `true` | Include the attachments you sent |
| `embedded` | query | boolean | no | `true` | Include embedded documents (requires a specific permission) |
| `base64` | query | boolean | no | `true` | Include the content in Base64 in addition to the `url` |
| `documents` | query | string | no | all | Comma-separated list with the desired types |

Types accepted in `documents`: `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

Full details in [Shipment Documents](api-documentos.md).

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

## Endpoint: update shipment

- **Method and route:** `PUT /v1/shipping/{shipping}`
- **When to use:** only in exceptional cases — linking the NF-e, attaching missing documents
  or informing the cost center.
- **Rule:** send `nfkey` and/or `attachments` and/or `cost_center_code`. **At least one** must be present.

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `nfkey` | string | conditional | 44 | 44 | NF-e key (44 numeric digits) |
| `cost_center_code` | integer | conditional | - | - | Code of the cost center released for your user |
| `attachments` | array | conditional | - | - | Files to attach |
| `attachments[].description` | string | **yes** if `attachments` is sent | - | 100 | Description of the document |
| `attachments[].base64` | string | **yes** if `attachments` is sent | - | - | Content in Base64. Formats: `PDF`, `XML`, `JPEG`, `PNG`. Max. **5 MB** per file |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Nota Fiscal", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

| Status | Message | Cause |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | None of the three fields was sent |
| `404` | `Remessa não encontrada!` | Invalid `{shipping}` |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | The shipment has already advanced too far in the flow |

## Endpoint: remove attachments

- **Method and route:** `DELETE /v1/shipping/{shipping}/docs`
- **When to use:** to delete an attachment sent by mistake.

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `attachments` | array | **yes** | 1 item | - | List of the attachments to remove |
| `attachments[].id` | integer | **yes** if `name` is not sent | 1 | - | Attachment ID (returned in `GET .../docs`) |
| `attachments[].name` | string | **yes** if `id` is not sent | - | 100 | Exact description of the attachment |

> Send **`id` OR `name`** per element, never both.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!",
  "data": {
    "deleted": [
      { "id": 123456, "name": "Nota Fiscal" }
    ],
    "failed": [
      { "id": 789, "name": null, "reason": "Nenhum documento encontrado!" }
    ]
  }
}
```

> The operation is **partial**: what was removed appears in `deleted` and what failed in `failed`,
> with the reason. Always check the `failed` array.

## Endpoint: cancel shipment

- **Method and route:** `DELETE /v1/shipping/{shipping}`
- **By channel:** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **When to use:** to request the cancellation. The processing is **asynchronous**.

| Field | Location | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **yes** | - | - | UUID, AWB or NF-e key |
| `channel_identifier_number` | path | string | **yes** (by-channel variant) | - | - | Commercial identifier |
| `channel_code` | path | integer | no | - | - | Channel code. Only **numeric** values are interpreted as a channel |
| `reason` | body | string | no | 10 | 255 | Reason for the cancellation, in free text |

> **Watch out for the route ambiguity:** `DELETE /v1/shipping/{value}/{number}` is interpreted as
> **cancellation by channel**. To cancel by shipment identifier always use
> `DELETE /v1/shipping/{shipping}` (a single segment).

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Customer requested the order cancellation"}'
```

```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

| Status | Message | Cause |
|---|---|---|
| `400` | `Remessa especificada não foi encontrada ou não é possivel mais cancelar a mesma` | Invalid identifier or shipment in a status that does not allow cancellation |
| `422` | Validation | `reason` with fewer than 10 or more than 255 characters |

> After requesting the cancellation, query the [status](api-rastreio.md): it goes through `cancelling`
> and, once confirmed by the carrier, moves to `cancelled`.

## Business rules

### Quotation eligibility
| Condition | Message when it fails |
|---|---|
| The quotation exists and belongs to the account | `Nenhuma cotação foi encontrada com os dados especificados!` |
| The quotation is not expired | `Esta cotação está expirada...` |
| The quotation has not been used before | `Cotação já foi utilizada em outra remessa!` |
| Same API version | `Não é possivel finalizar, pois seu quotation pode estar inválido...` |
| It is not a `simple` quotation of `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| The contract is in the quotation's list | `Contrato especificado não foi listado para esta cotação!` |

### Consistency between quotation and shipment
The address informed in the shipment must be compatible with the quotation one. Divergences return:
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
- `foreign` exists so that the `federal_tax_id` validation is done in the correct format
  (Brazilian document vs. foreign document).

### Carrier restrictions
- `Cotações e envios via FedEx entre Europa e Oriente Médio estão temporariamente suspensos pela transportadora.`
- For some contracts (e.g. DHL with a company sender in Brazil), the `nfkey` is required.
  Its absence returns `Chave da Nota não foi informada!!`.

### Automatic cancellation
Shipments that go **29 days or more** without a tracking event in an eligible status may be cancelled
automatically by the platform.

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Business rule (quotation, address, payment, tax document) | Read `message` and fix the scenario |
| `401` | Payment not authorized | Enable the payment type in the account |
| `403` | No permission for the action | Ask support for the permission |
| `404` | Shipment/contract not found | Check the identifier and the `carrier_code`/`slug` |
| `422` | Field validation or shipment in a non-changeable status | Fix the fields listed in `data` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- Always store the `quotation`: it is the most stable identifier of the shipment.
- Informing `channel_identifier_number` in the quotation is the most practical way to query and cancel
  using your own order number.
- To inform city, state and postal code **in the shipment** (and not in the quotation), use
  [Shipment V2](api-remessa-v2.md).
- For quotation + shipment + documents in **a single synchronous call**, see [Manifest](api-manifesto.md).
