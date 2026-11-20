# Shipment (v1)

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md) and [Quotation (v1)](api-cotacao-v1.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Turns a **v1 quotation** into a real shipment: generates the AWB, invoice, declaration and the other documents.
- It is the most complete module of the cycle: creation, status query, document query, update,
  attachment removal and cancellation.
- The status, documents, update and cancellation endpoints **also** serve shipments created in
  [Shipment V2](api-remessa-v2.md).
- Creation is **asynchronous**: the API registers the shipment and the dispatch to the carrier happens afterwards.

> ⚠️ **The shipment is an irreversible operation in the normal route.** Once created, it goes into processing
> at the carrier. The only later changes allowed are exceptional: linking the NF-e when it
> was not provided, attaching documents that were not sent on creation and providing/changing the cost center.

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
1. POST /v1/quotation                  → quotation + carriers[]
2. POST /v1/shipping                   → shipment received (asynchronous)
3. GET  /v1/shipping/{shipping}/status → follow until "shipped"
4. GET  /v1/shipping/{shipping}/docs   → download the AWB, invoice etc.
5. POST /v1/schedule                   → schedule pickup (optional)
```

1. [Create the v1 quotation](api-cotacao-v1.md) and check whether it is [eligible](#quotations-ineligible-for-shipment).
2. Send the shipment with the `quotation` and the chosen carrier (`carrier_code` **or** `slug`).
3. Creation is **asynchronous**: the `200` response means "we received it", not "we dispatched it".
4. Check the [status](api-rastreio.md) until it appears as `shipped`.
5. Download the [documents](api-documentos.md).
6. If necessary, [schedule the pickup](api-agendamento.md).

## Key concepts

### Shipment from a v1 quotation
- The v1 quotation has already brought the origin and destination **country, city, state and postal code**. That is why, in the v1 shipment, you
  only provide **who** sends and **who** receives (name, contact, street, tax documents).
- Location fields (`city`, `state`, `state_code`, `postal_code`, `country_code`) sent in
  `address_sender`/`address_receiver` are **ignored** in v1: the location is always the one from the quotation.
- A quotation generates **a single** shipment.

### The `{shipping}` identifier

In the documents, update, attachment removal and cancellation endpoints, the `{shipping}` path parameter
accepts **three formats**:

| Format | Example | Note |
|---|---|---|
| Quotation UUID | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Recommended form. Any value with a hyphen is treated as a UUID |
| NF-e key | `12345678901234567890123456789012345678901234` | Exactly 44 numeric digits |
| AWB number | `271249639538` | Shipments created in the **last 90 days** |

> The **status** endpoint only accepts UUID or AWB. Details in [Shipment Tracking](api-rastreio.md).

To query without storing any of these values, use the **by channel** endpoints, based on the
`channel_identifier_number` you provided in the quotation.

## Endpoint: create shipment

- **Method and route:** `POST /v1/shipping`
- **When to use:** after getting an eligible v1 quotation, within the validity period.

### Parameters — payload root

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `quotation` | string | **yes** | - | - | - | Quotation UUID. It must have been created in **v1** |
| `carrier_code` | integer | **yes** if `slug` is not sent | - | - | - | Contract code (`carriers[].code` from the quotation) |
| `slug` | string | **yes** if `carrier_code` is not sent | - | 20 | - | Text identifier of the contract (`carriers[].slug` from the quotation). When sent, it takes precedence over `carrier_code` |
| `address_sender` | object | **yes** | - | - | - | Sender (see table below) |
| `address_receiver` | object | **yes** | - | - | - | Receiver (see table below) |
| `payment` | object | **yes** | - | - | - | Payment method (see table below) |
| `type` | string | no | - | - | account preset or `Commercial` | Purpose of the shipment. Values: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | conditional | 44 | 44 | `null` | NF-e key. When provided, it must have **exactly 44 characters**. Required in some contracts (see [Business rules](#carrier-restrictions)) |
| `invoice_number` | string | no | 1 | 50 | internal code | Invoice number. If omitted, the internal code of the shipment is used |
| `shipment_reference` | string | no | 1 | 50 | internal ID | Shipment reference. If omitted, the internal ID of the shipment is used |
| `cost_center_code` | integer | no | - | - | account preset | Cost center code. It must exist and be authorized for your user. Empty/null receives the account's default cost center |
| `payment_notify_shipping_billing` | boolean | no | - | - | account preset | With `payment.type: afterward`, sends the shipment billing e-mail |
| `attachments` | array | no | - | - | `null` | Documents attached to the shipment (see table below) |
| `biller` | object | no | - | - | account preset | Biller displayed on the invoice instead of the sender (see table below) |

> **`carrier_code` or `slug`:** send **one** of the two. The provided contract must be in the
> `carriers[]` list returned by the quotation; otherwise the API responds `404` with
> `Contrato especificado não foi listado para esta cotação!`.

### Parameters — `address_sender` and `address_receiver`

Two forms, do not mix them:

**Form A — already registered address:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this form | Code of the registered address. **It must have the same country, state and city as the quotation** |

**Form B — manual address:**

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
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
| `federal_tax_id` | string | conditional | - | 20 | `null` | CPF/CNPJ or federal document. Required depending on the country (see [Tax documents](#required-tax-documents)) |
| `state_tax_id` | string | conditional | - | 20 | `null` | State registration. Required for Brazilians (`BR` with `foreign: false`) |
| `foreign` | boolean | no | - | - | `false` if the country is `BR`; `true` otherwise | Indicates whether the person is a foreigner. Defines how the `federal_tax_id` is validated in Brazil |

> The country, state, city and postal code are **not** provided here: they come from the quotation.
> If you need to provide them in the shipment, use [Shipment V2](api-remessa-v2.md).

### Parameters — `payment`

| Field | Type | Required | Values | Description |
|---|---|---|---|---|
| `payment.type` | string | **yes** | `billed`, `gateway`, `afterward` | Financial model of the shipment |

| Value | Meaning | Account requirement |
|---|---|---|
| `billed` | Billed: the amount goes into your periodic invoice | "Billed" payment option enabled |
| `gateway` | Immediate payment via gateway, in the currency defined in `currency_payment` | "Immediate" payment option enabled |
| `afterward` | Later payment, settled outside the automatic flow | - |

> A value outside the list returns `422`. A type not enabled for your account returns
> `401` with `Opção de pagamento não autorizado!`.

### Parameters — `attachments[]`

| Field | Type | Required | Max. | Description |
|---|---|---|---|---|
| `description` | string | **yes** | 100 | Name/description of the document (used later to find or remove the attachment) |
| `base64` | string | **yes** | - | File content encoded in Base64 |

| Limit | Value |
|---|---|
| Accepted formats | `PDF`, `XML`, `JPEG`, `PNG` |
| Total request size | **90 MB** |

> ⚠️ **On creation, invalid attachments are silently discarded:** a file in an unrecognized format or larger
> than the limit does not generate an error — the shipment is created without it. Check the stored attachments in
> [Documents](api-documentos.md). To attach later, with validation and error messages, use
> [update shipment](#endpoint-update-shipment) (limit of **5 MB** per file).

### Parameters — `biller`

When provided, this address appears on the invoice instead of the sender's address.
Send `address_code` **or** the manual fields. **If you are not going to use a biller, do not send the field** (or send it empty:
in that case the biller preconfigured in the account applies, if any).

| Field | Type | Min. | Max. | Description |
|---|---|---|---|---|
| `address_code` | integer | - | - | Registered address code |
| `country_code` | string | 2 | 2 | Country in ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | State name |
| `state_code` | string | - | 10 | State abbreviation |
| `postal_code` | string | - | 12 | Postal code/ZIP |
| `city` | string | - | 100 | City |
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
    "street": "Rua Principal",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  },
  "address_receiver": {
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

> `federal_tax_id` (valid CNPJ) and `state_tax_id` appear in the minimum payload because the sender is a Brazilian
> legal entity. For a sender from another country (except `MX` and `GR`), these fields can be omitted.

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
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
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
| `message` | string | Receipt confirmation |

> The response does **not** include the AWB or documents: the shipment was only received. Use the
> [tracking](api-rastreio.md) to follow it and the [documents](api-documentos.md) to download the files.
> The shipment identifier is the `quotation` itself.

### Creation-specific errors

| Status | Message | Cause / fix |
|---|---|---|
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Federal document missing where it is required (see [Tax documents](#required-tax-documents)) |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brazilian without `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ with incorrect check digit |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF with incorrect check digit |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` in Brazil with exempt `state_tax_id` |
| `400` | `Numero da nota fiscal é requerido` | The contract requires `nfkey` for this scenario |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` of a removed/inactive contract |
| `401` | `Opção de pagamento não autorizado!` | Payment type not enabled in your account |
| `403` | `Cotação Simples é inelegível para remessa!` | Use `type: advanced` or `items` in the quotation |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` exceeded — redo the quotation |
| `403` | `Has no permission!` | Your profile does not have permission to create shipments |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | `quotation` from another account |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` outside the quotation list |
| `404` | `O endereço especificado, não foi encontrado!` | `address_code` invalid or from another account |
| `406` | `Cotação já foi utilizada em outra remessa!` | A quotation generates **one** shipment |
| `406` | `O país :country é diferente do enviado na cotação!` | `address_code` with a country different from the quoted one |
| `406` | `O Estado :state é diferente do enviado na cotação!` | `address_code` with a state different from the quoted one |
| `406` | `A cidade :city é diferente do enviado na cotação!` | `address_code` with a city different from the quoted one |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | `quotation` nonexistent or created in v2 (comes in `data.quotation`) |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente)...` | Account locked to DDP with an unpaid DDU quotation |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` not authorized for you |

> When there is more than one tax document error, the messages come together in `message`, separated by ` | `.

## Endpoint: query shipment status

- **Method and route:** `GET /v1/shipping/{shipping}/status`
- **By channel:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

### Parameters

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `shipping` | path | string | **yes** | Quotation UUID or AWB |
| `channel_identifier_number` | path | string | **yes** (by channel variant) | Commercial identifier provided in the quotation |
| `channel_code` | path | integer | no | Sales channel code |

### Request example

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

> Response format, status groups and codes in [Shipment Tracking](api-rastreio.md).

## Endpoint: query documents

- **Method and route:** `GET /v1/shipping/{shipping}/docs`
- **By channel:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

### Parameters

| Parameter | Location | Type | Required | Default | Description |
|---|---|---|---|---|---|
| `shipping` | path | string | **yes** | - | UUID, NF-e key or AWB |
| `attachments` | query | boolean | no | `true` | Include the attachments sent by you (requires `base64=true`) |
| `embedded` | query | boolean | no | `true` | Include embedded documents (requires a specific permission and `base64=true`) |
| `base64` | query | boolean | no | `true` | Include the Base64 content in addition to the `url` |
| `documents` | query | string | no | all | Comma-separated list with the desired types |

Types accepted in `documents`: `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

### Request example

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

> Response format and details of each document in [Shipment Documents](api-documentos.md).

## Endpoint: update shipment

- **Method and route:** `PUT /v1/shipping/{shipping}`
- **When to use:** only in exceptional cases — linking the NF-e, attaching missing documents
  or providing/changing/removing the cost center.
- **Rule:** send `nfkey` and/or `attachments` and/or `cost_center_code`. **At least one** must be present.

### Parameters

| Field | Location | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **yes** | - | - | UUID, NF-e key or AWB |
| `nfkey` | body | string | conditional | 44 | 44 | NF-e key. **It is only stored if the shipment does not have an NF-e yet**; if it already has one, it is ignored |
| `cost_center_code` | body | integer | conditional | - | - | Code of the cost center authorized for your user. Sending `null` **removes** the cost center |
| `attachments` | body | array | conditional | - | - | Files to attach |
| `attachments[].description` | body | string | **yes** if `attachments` is sent | - | 100 | Document description |
| `attachments[].base64` | body | string | **yes** if `attachments` is sent | - | - | Base64 content. Formats: `PDF`, `XML`, `JPEG`, `PNG`. Max. **5 MB** per file |

### Minimum payload

```json
{
  "cost_center_code": 1
}
```

### Maximum payload

```json
{
  "nfkey": "12345678901234567890123456789012345678901234",
  "cost_center_code": 1,
  "attachments": [
    { "description": "Invoice", "base64": "JVBERi0xLjcK..." },
    { "description": "Product photo", "base64": "iVBORw0KGgoAAAANSUhEUg..." }
  ]
}
```

### Request example

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Invoice", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success` when the update was applied |
| `message` | string | Confirmation |

| Status | Message | Cause |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | None of the three fields was sent |
| `400` | `Status da remessa não permite atualização na NFE.` | The shipment is already past `shipped` (or is canceled/with error) |
| `400` | `Erro ao salvar arquivo!` | Invalid `base64` |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Attachment above 5 MB |
| `404` | `Esta remessa não foi encontrada!` | `{shipping}` invalid or from another account |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Shipment canceled or with a closed cycle |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Attachment format not accepted |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | Malformed XML |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Cost center not authorized |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Unexpected error |

> The changes are applied in the order NF-e → cost center → attachments. If an attachment fails, the NF-e and the cost
> center sent in the same call **have already been stored**.

## Endpoint: remove attachments

- **Method and route:** `DELETE /v1/shipping/{shipping}/docs`
- **When to use:** to delete an attachment sent by mistake.

### Parameters

| Field | Location | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **yes** | - | - | UUID, NF-e key or AWB |
| `attachments` | body | array | **yes** | 1 item | - | List of attachments to remove |
| `attachments[].id` | body | integer | **yes** if `name` is not sent | 1 | - | Attachment ID (returned in `GET .../docs`) |
| `attachments[].name` | body | string | **yes** if `id` is not sent | - | 100 | Exact description of the attachment |

> Send **`id` OR `name`** per item, never both.

### Minimum payload

```json
{
  "attachments": [
    { "id": 123456 }
  ]
}
```

### Maximum payload

```json
{
  "attachments": [
    { "id": 123456 },
    { "name": "Invoice" }
  ]
}
```

### Request example

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Invoice"}
    ]
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!",
  "data": {
    "deleted": [
      { "id": 123456, "name": "Invoice" }
    ],
    "failed": [
      { "id": 789, "name": null, "reason": "Nenhum documento encontrado!" }
    ]
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `data.deleted[]` | array | Removed attachments (`id`, `name`) |
| `data.failed[]` | array | Attachments not removed, with the reason in `reason` |

> The operation is **partial**: what was removed appears in `deleted` and what failed in `failed`,
> with the reason. Always check the `failed` array.

| Status | Message | Cause |
|---|---|---|
| `400` | `Remessa não encontrada!` | UUID without a shipment |
| `404` | `Esta remessa não foi encontrada!` | AWB/NF-e not found or from another account |
| `422` | Validation | Empty `attachments`, or item without `id`/`name` |

## Endpoint: cancel shipment

- **Method and route:** `DELETE /v1/shipping/{shipping}`
- **By channel:** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **When to use:** to request the cancellation. The response confirms the request; completion may be asynchronous.

### Parameters

| Field | Location | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **yes** | - | - | UUID, NF-e key, AWB **or** `channel_identifier_number` |
| `channel_identifier_number` | path | string | **yes** (by channel variant) | - | - | Commercial identifier |
| `channel_code` | path | integer | no | - | - | Channel code. Only **numeric** values are interpreted as a channel |
| `reason` | body | string | no | 10 | 255 | Cancellation reason, in free text |

> **How the route is interpreted:**
> - `DELETE /v1/shipping/{value}/{number}` is always **cancellation by channel** (`channel_identifier_number` + `channel_code`).
> - `DELETE /v1/shipping/{value}` first looks for a shipment whose `channel_identifier_number` is `{value}`;
>   if none is found, it treats `{value}` as UUID, NF-e key or AWB.

### Minimum payload

No body:

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer YOUR_TOKEN'
```

### Maximum payload

```json
{
  "reason": "Customer requested the order cancellation"
}
```

### Request example

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/PED-2026-0001/1 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"reason":"Customer requested the order cancellation"}'
```

### Response example

```json
{
  "status": "success",
  "message": "Solicitação de cancelamento da remessa 9c06fd75-d5f4-40eb-a1a8-f12086f49944 recebida. Em breve será processada; consulte o status para acompanhar."
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success` when the cancellation request was accepted |
| `message` | string | Confirmation, with the shipment UUID |

| Status | Message | Cause |
|---|---|---|
| `400` | `Esta remessa não foi encontrada!` | Invalid identifier or shipment from another account |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | The current status does not allow cancellation (see table below) |
| `422` | Validation | `reason` with fewer than 10 or more than 255 characters; nonexistent `channel_code` |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Unexpected error |

**When the cancellation is accepted:**

| Current shipment status | Result |
|---|---|
| `received`, `processing_payment`, `paid_out` | Canceled immediately (`cancelled`) |
| `shipped` | Goes to `cancelling` and is canceled at the carrier; some contracts cancel directly (`cancelled`) |
| `error` without a shipment registered at the carrier | Canceled immediately (`cancelled`) |
| `processing_shipment` | Accepted only in specific contracts; refused in the others |
| Others (`collecting`, `in_transit`, `delivered`, `cancelled`...) | Refused with `Remessa não atende requisitos para cancelamento :id !` |

> After requesting the cancellation, check the [status](api-rastreio.md): it goes through `cancelling`
> and, when confirmed by the carrier, goes to `cancelled`.

## Quotations ineligible for shipment

Before registering the shipment, the API checks the quotation in this order. The first condition that fails interrupts
the creation:

| # | Check | Status | Message |
|---|---|---|---|
| 1 | `quotation` exists and was created in **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | The quotation belongs to your account | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Account locked to DDP: DDU quotation must be paid | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | Tax documents of sender and receiver | `400` | See [Required tax documents](#required-tax-documents) |
| 5 | Quotation not used yet | `406` | `Cotação já foi utilizada em outra remessa!` |
| 6 | It is not `simple` of `package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 7 | Quotation within its validity | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 8 | Contract is in the quotation list | `404` | `Contrato especificado não foi listado para esta cotação!` |
| 9 | Payment type enabled for the account | `401` | `Opção de pagamento não autorizado!` |
| 10 | NF-e provided when the contract requires it | `400` | `Numero da nota fiscal é requerido` |

> To know, **before** shipping, whether a quotation has already been used, query
> `GET /v1/quotation/{quotation}` and check `data.finished`. Details of the eligible scenarios in
> [Quotation (v1)](api-cotacao-v1.md#quotations-ineligible-for-shipment).

## Business rules

### Consistency between quotation and shipment
- With a manual address (Form B), the location is always the one from the quotation.
- With `address_code` (Form A), the registered address must have **the same country, state and city** as the quotation.
  Divergences return `406`:
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Required tax documents
| Address country | `type` | `foreign` | `federal_tax_id` | `state_tax_id` |
|---|---|---|---|---|
| `BR` | `pf` | `false` (default) | **Required** — valid CPF | **Required** |
| `BR` | `pj` | `false` (default) | **Required** — valid CNPJ | **Required** |
| `BR` | any | `true` | **Required** (foreign document, without CPF/CNPJ validation) | Optional |
| `MX`, `GR` | any | - | **Required** | Optional |
| Others | any | - | Optional | Optional |

- For `type: pf` in Brazil, the state registration cannot be `ISENTO`/`ISENTA`.
- For `type: pj` in any country, a `federal_tax_id` in CNPJ format is validated as a CNPJ.
- With `address_code`, the tax documents are read from the address registration and go through the same rules.

### Carrier restrictions
- For DHL contracts with a **legal entity sender in Brazil** and an object other than a document, the `nfkey` is
  required. Its absence returns `400` with `Numero da nota fiscal é requerido`.

### Presets applied to the shipment
| Preset | Effect |
|---|---|
| Cost center | Fills `cost_center_code` when it is not sent, or is sent empty/null |
| Shipment purpose | Fills `type` when it is not sent |
| Biller | Fills `biller` when it is not sent (or is sent empty) and the account displays a biller |
| Billing notification | Fills `payment_notify_shipping_billing` when it is not sent |

### Automatic cancellation
Shipments that remain **29 days** without a tracking number, in a status prior to dispatch, may be canceled
automatically by the platform. Details in [Shipment Tracking](api-rastreio.md).

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | Tax document, required NF-e, invalid contract by `slug`, cancellation not allowed | Read `message` and fix the scenario |
| `401` | Payment not authorized | Enable the payment type in the account |
| `403` | Simple/expired quotation, or no permission for the action | Redo the quotation / request the permission from support |
| `404` | Quotation from another account, shipment/contract/address not found | Check the identifier and the `carrier_code`/`slug` |
| `406` | Quotation already used or `address_code` incompatible with the quotation | Make a new quotation / use the quoted address |
| `422` | Field validation, quotation from another version, DDP-only account, invalid attachment | Fix the fields listed in `data` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- Always store the `quotation`: it is the most stable identifier of the shipment.
- Providing `channel_identifier_number` in the quotation is the most practical way to query and cancel
  using your own order number.
- To provide city, state and postal code **in the shipment** (and not in the quotation), use
  [Shipment V2](api-remessa-v2.md). Differences in [Quotation & Shipment: v1 vs v2](api-comparativo-cotacao-remessa.md).
- For quotation + shipment + documents in **a single synchronous call**, see [Manifest](api-manifesto.md).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
