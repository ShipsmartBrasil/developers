# Pickup Scheduling

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md) and [Shipment (v1)](api-remessa-v1.md)

## Overview
- Asks the carrier to **pick up** the package at your address, instead of you taking it to a drop-off
  point.
- Allows you to query the scheduling status and to cancel it with a standardized reason.
- It works for shipments created in **v1 and v2**.

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

## Prerequisites

Before scheduling, confirm that:

1. The shipment is already in the **`shipped`** status ([tracking](api-rastreio.md)) — that is, it has already been processed at the
   carrier and the documents exist.
2. The carrier of the chosen contract **has an integrated pickup service**.
3. The shipment **does not have** another scheduling in progress.

## Usage flow

```
1. GET  /v1/shipping/{shipping}/status   → confirm the "shipped" status
2. POST /v1/schedule                     → create the scheduling
3. GET  /v1/schedule/{shipping}          → follow until the "success" status
4. DELETE /v1/schedule/{shipping}/{reason} → cancel (if necessary)
```

## Conventions
- Dates in the `Y-m-d` format (e.g. `2026-03-10`).
- Times in the `H:i:s` format (e.g. `08:00:00`).
- The `{shipping}` identifier accepts the **quotation UUID** or the **AWB number** (shipments from the
  last 90 days). Unlike the shipment endpoints, **the NF-e key is not accepted here**.

## Endpoint: create scheduling

- **Method and route:** `POST /v1/schedule`
- **When to use:** after the shipment reaches `shipped`.

### Parameters

| Field | Type | Required | Min. | Max. | Format | Description |
|---|---|---|---|---|---|---|
| `shipping` | string | **yes** if `awb_number` is not sent | - | - | UUID | Quotation/shipment code |
| `awb_number` | string | **yes** if `shipping` is not sent | - | 50 | - | AWB number (shipments from the last 90 days) |
| `date` | string | **yes** | - | - | `Y-m-d` | Pickup date. It must be **later than today** |
| `initial_time` | string | **yes** | - | - | `H:i:s` | Start of the pickup window |
| `final_time` | string | **yes** | - | - | `H:i:s` | End of the pickup window |
| `instruction` | string | **yes** | 5 | 80 | - | Instruction for the driver (e.g. where to pick up, who to talk to) |
| `address` | object | no | - | - | - | Alternative pickup address (see table below) |

> Send **`shipping` or `awb_number`**, not both.

### Parameters — `address` (alternative pickup address)

Only send it if the pickup happens at an address **different** from the shipment's sender.
**The address must be in the same country as the sender.**

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `street` | string | **yes** if `address` is sent | 5 | 70 | Street |
| `city` | string | **yes** if `address` is sent | 5 | 35 | City |
| `name` | string | **yes** if `address` is sent | 5 | 35 | Name of the contact at the location |
| `email` | string | **yes** if `address` is sent | - | 100 | Contact e-mail |
| `phone` | string | **yes** if `address` is sent | 10 | 25 | Contact phone |
| `postal_code` | string | conditional | - | 12 | Postal/ZIP code. **Required for pickups in Brazil** |
| `state_code` | string | no | - | 10 | State abbreviation. It must belong to the sender's country |
| `number` | string | conditional | - | 6 | Building number. **Required for pickups in Brazil** |
| `district` | string | conditional | - | 30 | District. **Required for pickups in Brazil** |
| `complement` | string | no | - | 20 | Complement |
| `phone_extension` | string | no | - | 5 | Extension |
| `federal_tax_id` | string | conditional | - | 20 | CPF/CNPJ. **Required and validated for pickups in Brazil** |

> When `address` is omitted, the platform resolves the address automatically: first the pickup preset
> of your account and, in its absence, the shipment's sender address.
>
> In some account types (individual and single-DC company), the alternative address is **ignored** and the pickup
> always uses the sender's address.

### Minimum payload

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Pick up at the reception, ask for Carlos"
}
```

### Maximum payload

```json
{
  "awb_number": "271249639538",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Pick up at warehouse 2, side gate, ask for Carlos",
  "address": {
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Warehouse 2",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

### Request example

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2026-03-10",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Pick up at the reception, ask for Carlos"
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> The response confirms the **receipt**. The carrier's confirmation is asynchronous: query the
> scheduling until the status turns into `success` and the `confirmation_code` appears.

### Date and window rules

| Rule | Detail | Message when it fails |
|---|---|---|
| Future date | `date` must be **later than the current date** | `A data de agendamento não pode ser menor que a data Atual` |
| No weekends | Saturday and Sunday are not accepted | `Coleta não pode ser realizada em um final de semana` |
| No national holidays | For pickups in Brazil | `Coleta não pode ser realizada durante um feriado nacional` |
| 7 business day limit | The date cannot exceed 7 business days from today | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Brazilian national holidays considered: **01/01, 21/04, 01/05, 07/09, 12/10, 02/11, 15/11, 20/11 and 25/12**.

## Endpoint: query scheduling

- **Method and route:** `GET /v1/schedule/{shipping}`

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `shipping` | path | string | **yes** | Quotation UUID or AWB number (last 90 days) |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Pick up at the reception, ask for Carlos",
    "status": {
      "code": "processing",
      "description": "Agendamento da Remessa Recebida. Em breve será processada!",
      "reason": null
    }
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `data.confirmation_code` | string | Pickup confirmation number at the carrier. Filled in when the status reaches `success` |
| `data.date` | string | Pickup date (`Y-m-d`) |
| `data.inicial_time` | string | Start of the window (`H:i:s`). ⚠️ **The key is `inicial_time`** in the response, while in the request the field is `initial_time` |
| `data.final_time` | string | End of the window (`H:i:s`) |
| `data.instruction` | string | Instruction informed at creation |
| `data.status.code` | string | Scheduling status code (see table) |
| `data.status.description` | string | Description translated according to the `lang` header |
| `data.status.reason` | string \| null | Cancellation reason code (`001`–`008`). `null` while there is no cancellation |

### Scheduling status

| `code` | Meaning |
|---|---|
| `processing` | Scheduling received, waiting to be sent to the carrier |
| `success` | **Pickup confirmed by the carrier.** `confirmation_code` available |
| `canceling` | Cancellation requested, waiting to be processed |
| `terminating` | Cancellation being processed at the carrier |
| `canceled` | Cancellation completed |
| `error` | Failure in the scheduling or in the cancellation |

### Processing type

The platform classifies the scheduling automatically, with no field in the payload:

| Type | When | Behavior |
|---|---|---|
| `automatico` | Pickup for the next day (up to 1 day) | Sent to the carrier immediately |
| `agendado` | Pickup with more than 1 day of lead time | Sent to the carrier 1 day before the pickup |

> That is why a future scheduling may stay in `processing` for days. This is expected.

## Endpoint: cancel scheduling

- **Method and route:** `DELETE /v1/schedule/{shipping}/{reason}`
- **When to use:** when the pickup has already been **confirmed** (`success` status) and needs to be called off.

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `shipping` | path | string | **yes** | Quotation UUID or AWB number (last 90 days) |
| `reason` | path | string | **yes** | Reason code (see table). Sent to the carrier |

### Cancellation reasons

| Code | Reason |
|---|---|
| `001` | Package not ready |
| `002` | Rates too high |
| `003` | Transit time too slow |
| `004` | Take to service center or drop box |
| `005` | Commitment time not met |
| `006` | Reason not given |
| `007` | Other |
| `008` | Pickup modified |

> These codes are standardized by the carriers and passed on to them without translation.
> Use `006` or `007` when none of the specific ones applies.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Cancellation rules

| Rule | Message when it fails |
|---|---|
| The `reason` must be between `001` and `008` | `Motivo do cancelamento não é compativel` |
| The scheduling must exist | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| The shipment must be in `shipped` | same as above |
| The scheduling must be in `success` (already confirmed by the carrier) | same as above |

> After the request, the status moves to `canceling` with the description
> `Solicitação de cancelamento recebida! Em breve será processada!`, then `terminating` and, finally,
> `canceled` (or `error`).

## Consolidated business rules

| Rule | Detail |
|---|---|
| Shipment status | The shipment must be in `shipped` |
| Single scheduling | It is not possible to have two active schedulings on the same shipment |
| Carrier | The service must support integrated pickup |
| Date | Future, business day, no national holiday (BR), up to 7 business days |
| BR address | Requires `postal_code`, `number`, `district` and a valid `federal_tax_id` (CPF or CNPJ) |
| Address country | The `state_code` must belong to the country of the shipment's sender |
| Ownership | It is only possible to schedule/query/cancel shipments of your own account |
| AWB | Resolves only shipments from the **last 90 days** |

## Common errors

| Status | Message | How to fix it |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Query the existing scheduling before creating another one |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Wait for the `shipped` status |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Take the package to a drop-off point |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Use a future date |
| `400` | `Coleta não pode ser realizada em um final de semana` | Choose a business day |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Choose another business day |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Reduce the lead time |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Complete `number` and `district` (pickups in Brazil) |
| `400` | `O atributo federal_tax_id não é válido` | Inform a valid CPF or CNPJ |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Use a `state_code` from the sender's country |
| `400` | `Motivo do cancelamento não é compativel` | Use a code between `001` and `008` |
| `403` | No permission | Ask support for the scheduling permission |
| `404` | `Agendamento não foi encontrado para esta remessa` | Check the identifier or create the scheduling |
| `422` | Validation | Fix the date/time/instruction according to the formats |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Retry with backoff; if it persists, contact support |

## Final notes
- Always query the scheduling before creating or cancelling: this avoids redundant requests and the
  `já tem um agendamento em progresso` error.
- Mind the naming difference: **`initial_time` in the request** and **`inicial_time` in the response**.
- Future schedulings stay in `processing` until 1 day before the pickup. This is not an error.
