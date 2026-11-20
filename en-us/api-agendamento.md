# Pickup Scheduling

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md) and [Shipment (v1)](api-remessa-v1.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Asks the carrier to **pick up** the package at your address, instead of you taking it to a drop-off
  point.
- Allows you to check the progress of the scheduling and cancel it with a standardized reason.
- Works for shipments created in **v1 and v2** (the scheduling endpoints exist only in `/v1`).
- The request is **asynchronous**: the API registers the request and the submission to the carrier happens in the
  background. The confirmation arrives later, in the scheduling query.
- Each shipment accepts **a single scheduling** through the API.

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

> `Content-Type` is only required in the `POST`. Each operation requires its own permission in your profile
> (create, query and cancel scheduling). Without the permission, the response is `403` with `Has no permission!`.

## Usage flow

```
1. GET    /v1/shipping/{shipping}/status    → confirm that the CURRENT status is "shipped"
2. POST   /v1/schedule                      → register the scheduling (status "processing")
3. GET    /v1/schedule/{shipping}           → follow up until the status is "success"
4. DELETE /v1/schedule/{shipping}/{reason}  → cancel, if necessary (only with status "success")
```

### Before scheduling, confirm

| # | Condition | How to check |
|---|---|---|
| 1 | The **current** status of the shipment is **`shipped`** (processed at the carrier, documents available). After the shipment moves forward (e.g., in transit), scheduling is no longer accepted | [Tracking](api-rastreio.md) |
| 2 | The carrier of the shipment contract **has integrated pickup** (see [Carriers with integrated pickup](#carriers-with-integrated-pickup)) | Contract chosen in the shipment |
| 3 | The shipment **does not yet have** any scheduling — not even a canceled one or one with an error | `GET /v1/schedule/{shipping}` must respond `404` |
| 4 | The shipment was created by the **same user** as the token | - |
| 5 | Your account does **not** have automatic pickup enabled in the presets (see [Automatic pickup via account presets](#automatic-pickup-via-account-presets)) | Account dashboard |

## Key concepts

### Conventions

| Item | Format | Example |
|---|---|---|
| Pickup date | `AAAA-MM-DD` (YYYY-MM-DD, exactly this format) | `2026-03-10` |
| Window times | `HH:MM:SS` (with seconds) | `08:00:00` |
| "Current" date | Considers the platform time zone (Brasília time) | - |
| `{shipping}` identifier | **Quotation UUID** or **AWB number** (shipments created in the last 90 days) | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` |

> Unlike the shipment endpoints, **the NF-e key is not accepted** as an identifier in scheduling.
> An identifier that contains a hyphen (`-`) is always treated as a UUID; without a hyphen, it is treated as an AWB.

### Carriers with integrated pickup

| Carrier | Scheduling via API | Cancellation via API |
|---|---|---|
| DHL | yes | yes |
| FedEx | yes | yes |
| UPS | yes | yes |
| SkyPostal | yes | yes |
| CTBI Log | yes | **no** (the cancellation request ends in `error`; contact support) |
| GLS | yes | **no** (the cancellation request ends in `error`; contact support) |

> List in effect on the date of this version of the guide; it may be expanded. For any other carrier,
> the `POST` responds `400` with `Esta transportadora não possui serviço de coleta integrado`.

### Pickup address

The pickup address is defined in one of these ways. **Do not mix them**: either you send `address`, or you let
the platform resolve it.

| Form | How to send | Address used |
|---|---|---|
| **Form A — automatic** | Omit `address` (or send `null` / `{}`) | 1st the pickup preset of your account for the sender country (if enabled and with a registered address); 2nd the sender address of the shipment |
| **Form B — alternative address** | Send the complete `address` object | The informed address, which must be **in the same country as the sender** |

> In **individual** and **single-DC legal entity (PJ mono-CD)** accounts, the `address` object is **ignored** and the pickup always uses
> the sender address of the shipment. Only the field format (`422`) is checked; the address
> rules (country, postal code, document) are not applied.

### Scheduling status

| `code` | Meaning | Next step |
|---|---|---|
| `processing` | Scheduling registered, awaiting submission/confirmation from the carrier | Wait and query again |
| `success` | **Pickup confirmed by the carrier.** `confirmation_code` available | Have the package ready within the informed window |
| `canceling` | Cancellation requested, awaiting submission to the carrier | Wait |
| `terminating` | Cancellation sent to the carrier, awaiting confirmation | Wait |
| `canceled` | Cancellation completed | - |
| `error` | Failure in the scheduling or in the cancellation. The reason comes in `status.description` | Read the description and contact support |

```
POST                         DELETE
 │                             │
 ▼                             ▼
processing ──► success ──► canceling ──► terminating ──► canceled
 │                             │              │
 └──► error                    └──► error     └──► error
```

### Processing type

The platform classifies the scheduling automatically by the informed date. There is no field in the payload.

| Type | When | Behavior |
|---|---|---|
| Immediate | `date` = tomorrow | Sent to the carrier in the next processing cycle |
| Scheduled | `date` the day after tomorrow or later | Sent to the carrier **1 day before** the pickup date |

> That is why a scheduled pickup remains in `processing` for a few days. This is expected.
> Example: a request made on Friday for the following Monday is **scheduled** (sent on Sunday).

### Automatic pickup via account presets

If your account has **automatic pickup** enabled in the presets (configured in the dashboard), the
platform creates the scheduling by itself as soon as the shipment documents are generated, without an API call:

| Item | Value applied |
|---|---|
| Date | Next business day (for a sender in Brazil, it also skips a national holiday) |
| Window | `08:00:00` to `18:00:00` |
| Instruction | The instruction registered in the preset |
| Address | Same rule as Form A (pickup preset and, failing that, the sender) |

> In this case, a `POST /v1/schedule` for the same shipment responds
> `400 Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso`.
> Use only `GET /v1/schedule/{shipping}` to follow up.

### Cancellation reasons

| Code | Reason |
|---|---|
| `001` | Package not ready (the package is not ready) |
| `002` | Rates too high (the rates are too high) |
| `003` | Transit time too slow (the transit time is too slow) |
| `004` | Take to service center or drop box (it will be taken to a drop-off point) |
| `005` | Commitment time not met (the agreed time was not met) |
| `006` | Reason not given (no reason provided) |
| `007` | Other (another reason) |
| `008` | Pickup modified (the pickup was modified) |

> These codes are standardized by the carriers and passed on to them without translation.
> Use `006` or `007` when none of the specific ones applies. Always send the **3 digits** (`001`, not `1`).

## Endpoint: create scheduling

- **Method and route:** `POST /v1/schedule`
- **When to use:** with the shipment in the current status `shipped`, to request the pickup of the package.

### Parameters

| Field | Type | Required | Min. | Max. | Format / values | Description |
|---|---|---|---|---|---|---|
| `shipping` | string | **yes** if `awb_number` is not sent | - | - | UUID of an existing quotation | Quotation/shipment code |
| `awb_number` | string | **yes** if `shipping` is not sent | - | 50 | - | AWB number (shipments created in the last 90 days) |
| `date` | string | **yes** | - | - | `AAAA-MM-DD` | Pickup date. See [Date rules](#date-rules) |
| `initial_time` | string | **yes** | - | - | `HH:MM:SS` | Start of the pickup window |
| `final_time` | string | **yes** | - | - | `HH:MM:SS` | End of the pickup window |
| `instruction` | string | **yes** | 5 | 80 | - | Instruction for the driver (where to pick up, who to talk to) |
| `address` | object \| null | no | - | - | - | Alternative pickup address (Form B). See table below |

> Send **`shipping` or `awb_number`**, not both. If both are sent, `awb_number` prevails.
>
> The API does **not** compare `initial_time` with `final_time` nor validate business hours. Make sure the end is
> after the start and that the window is accepted by the carrier; otherwise, the scheduling may
> end in `error`.

### Parameters — `address` (alternative pickup address)

Send it only if the pickup is at an address **different** from the automatic one.

**Attention: when `address` is sent, all 12 keys below must be present in the object.**
A missing key makes the request fail with `500`. When you do not have the data, use `null` in the fields that
accept null and `""` (empty text) in `complement` and `phone_extension`.

| Field | Type | Accepts `null` | Min. | Max. | Rule for a sender in Brazil | Description |
|---|---|---|---|---|---|---|
| `street` | string | no | 5 | 70 | - | Street |
| `number` | string | yes | - | 6 | **Required** (cannot be empty) | Building number |
| `district` | string | yes | - | 30 | **Required** (cannot be empty) | District/neighborhood |
| `complement` | string | no (use `""`) | - | 20 | - | Address complement |
| `city` | string | no | 5 | 35 | - | City |
| `state_code` | string | yes | - | 10 | - | State abbreviation. It must belong to the sender country |
| `postal_code` | string | yes | - | 12 | - | Postal code/ZIP. If `state_code` comes empty, the state is discovered from it |
| `name` | string | no | 5 | 35 | - | Name of the on-site contact |
| `email` | string | no | - | 100 | - | Contact e-mail (valid e-mail format) |
| `phone` | string | no | 10 | 25 | - | Contact phone |
| `phone_extension` | string | no (use `""`) | - | 5 | - | Phone extension |
| `federal_tax_id` | string | yes | - | 20 | **Required: valid CPF or CNPJ** | Document of the person responsible for the site |

How the country and the state are checked:

| Situation | Behavior |
|---|---|
| `state_code` informed | It must be a state of the sender country (or the country code itself). Otherwise: `Endereço enviado não pertence ao pais da remessa` |
| `state_code` empty and `postal_code` informed (BR sender) | The state is obtained from the postal code (CEP). Postal code not found: `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| `state_code` empty and `postal_code` informed (other countries) | The state is obtained from the postal code. Not found: `Informações de endereço de coleta insuficientes ou inválidas` |
| `state_code` and `postal_code` empty | There is no check. We recommend always sending both to avoid rejection by the carrier |

### Minimum payload

Form A with `shipping` (UUID):

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Pick up at the reception"
}
```

Form A with `awb_number`:

```json
{
  "awb_number": "1234567890",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Pick up at the reception"
}
```

### Maximum payload

Form B (alternative address, sender in Brazil), with all accepted fields:

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Pick up at warehouse 2, side gate, ask for Carlos",
  "address": {
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Warehouse 2",
    "city": "Cidade Exemplo",
    "state_code": "SP",
    "postal_code": "01000000",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

> To use the AWB, replace `shipping` with `awb_number`. The `federal_tax_id` in the example is fictitious: in
> shipments from Brazil, the API requires a valid CPF or CNPJ.

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
    "instruction": "Pick up at the reception"
  }'
```

### Response example

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> The response only confirms the **registration** of the request. The carrier confirmation is asynchronous:
> query the scheduling until the status becomes `success` and the `confirmation_code` appears.

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success` when the scheduling was registered |
| `message` | string | Message translated according to the `lang` header |

> There is no `data` in this response. To get details, use [query scheduling](#endpoint-query-scheduling).

### Date rules

The rules are checked in this order; the first one that fails defines the message (`400`).

| Rule | Detail | Message when it fails |
|---|---|---|
| Future date | `date` must be **after today** (today is not accepted) | `A data de agendamento não pode ser menor que a data Atual` |
| No weekends | Saturday and Sunday are not accepted, for any country | `Coleta não pode ser realizada em um final de semana` |
| No national holidays | Only when the sender is in Brazil | `Coleta não pode ser realizada durante um feriado nacional` |
| 7 business day limit | The date cannot go beyond the 7th business day after today (for a sender in Brazil, national holidays do not count as business days) | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Brazilian national holidays considered (fixed dates, DD/MM): **01/01, 21/04, 01/05, 07/09, 12/10, 02/11,
15/11, 20/11 and 25/12**. Movable holidays (e.g., Carnival, Good Friday, Corpus Christi) and state/municipal
holidays are **not** blocked by the API — avoid these dates on your own.

### Order of validations

| # | Check | Status | Message |
|---|---|---|---|
| 1 | Field format | `422` | Per-field messages in `data` |
| 2 | AWB found (if `awb_number` was sent) | `400` | `Agendamento não foi encontrado para esta remessa` |
| 3 | Date after today | `400` | `A data de agendamento não pode ser menor que a data Atual` |
| 4 | Shipment without previous scheduling | `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` |
| 5 | Existing shipment with current status `shipped` | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 6 | Carrier with integrated pickup | `400` | `Esta transportadora não possui serviço de coleta integrado` |
| 7 | Date rules (weekend, holiday, 7 business days) | `400` | See [Date rules](#date-rules) |
| 8 | Shipment created by the token user | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 9 | Alternative address (if sent) | `400` | See [Common errors](#common-errors) |

## Endpoint: query scheduling

- **Method and route:** `GET /v1/schedule/{shipping}`
- **When to use:** to follow up on the scheduling after the `POST` and before canceling.

### Parameters

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `shipping` | path | string | **yes** | Quotation UUID or AWB number (shipments from the last 90 days) |

### Minimum payload

No body. Only the identifier in the path:

```
GET /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944
```

### Maximum payload

There are no other parameters. The alternative is to use the AWB instead of the UUID:

```
GET /v1/schedule/1234567890
```

### Request example

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

### Response example

`200 OK`

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Pick up at the reception",
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
| `data.confirmation_code` | string \| null | Pickup confirmation number at the carrier. It is only final when `status.code` is `success`; before that it may come as `null` or as a provisional value (e.g., `000000`) |
| `data.date` | string | Pickup date (`AAAA-MM-DD`) |
| `data.inicial_time` | string | Start of the window (`HH:MM:SS`). ⚠️ **The key is `inicial_time`** in the response, while in the request the field is `initial_time` |
| `data.final_time` | string | End of the window (`HH:MM:SS`) |
| `data.instruction` | string | Instruction informed at creation (or the preset one, for automatic pickup) |
| `data.status.code` | string | Scheduling status code (see [Scheduling status](#scheduling-status)). Use this field in your logic |
| `data.status.description` | string | Informational text of the last event. **It is not translated in the query**: it may come in the language in which the request was made, in English (e.g., `Pickup Scheduled Successfully!`, `Pickup Cancelled Successfully!`) or with the message returned by the carrier, in case of `error` |
| `data.status.reason` | string \| null | Cancellation reason code (`001`–`008`). `null` while there is no cancellation request |

## Endpoint: cancel scheduling

- **Method and route:** `DELETE /v1/schedule/{shipping}/{reason}`
- **When to use:** when the pickup has already been **confirmed** (status `success`) and needs to be called off.

### Parameters

| Parameter | Location | Type | Required | Values | Description |
|---|---|---|---|---|---|
| `shipping` | path | string | **yes** | - | Quotation UUID or AWB number (shipments from the last 90 days) |
| `reason` | path | string | **yes** | `001` to `008` | Reason code (see [Cancellation reasons](#cancellation-reasons)). Passed on to the carrier |

### Minimum payload

No body. Identifier and reason in the path:

```
DELETE /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/006
```

### Maximum payload

There are no other parameters. The alternative is to use the AWB instead of the UUID:

```
DELETE /v1/schedule/1234567890/001
```

### Request example

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

### Response example

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success` when the cancellation request was registered |
| `message` | string | Message translated according to the `lang` header |

> After the request, the status changes to `canceling` with the description
> `Solicitação de cancelamento recebida! Em breve será processada!`, then `terminating` and, finally,
> `canceled` (or `error`). Query the scheduling to follow up.

### Cancellation rules

| Rule | Status | Message when it fails |
|---|---|---|
| The `reason` must be one of the codes `001` to `008` | `422` | `O campo reason não contém um valor válido.` (in `data.reason`) |
| The AWB must be found (if used in the path) | `400` | `Agendamento não foi encontrado para esta remessa` |
| The scheduling must exist | `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| The shipment must still have the current status `shipped` | `400` | same as above |
| The scheduling must be in `success` (already confirmed). In `processing`, `canceling`, `terminating`, `canceled` or `error`, the cancellation is refused | `400` | same as above |

## Business rules

| Rule | Detail |
|---|---|
| Shipment status | The **current** status must be `shipped`, both to schedule and to cancel |
| Single scheduling | Each shipment accepts a single scheduling through the API. A canceled scheduling or one with an error **also** prevents a new `POST`; to reschedule, contact support |
| Carrier | The shipment service must have integrated pickup. Integrated cancellation only for DHL, FedEx, UPS and SkyPostal |
| Date | After today, business day, no fixed national holiday (BR sender), up to 7 business days |
| Window | `initial_time` and `final_time` in the `HH:MM:SS` format; there is no validation of order or of business hours |
| Alternative address | All 12 keys present; in Brazil it requires `number`, `district` and a valid `federal_tax_id` (CPF or CNPJ) |
| Address country | Always the sender country of the shipment. The `state_code` must belong to that country |
| Account type | Individual and single-DC legal entity (PJ mono-CD): `address` ignored; pickup at the sender address |
| Automatic pickup | With the automatic pickup preset active, the scheduling is created by the platform and the manual `POST` is refused |
| Ownership | Schedule and query only shipments created by the token user |
| AWB | Resolves only shipments created in the **last 90 days** (the most recent one with that number) |
| Processing | Immediate for pickup tomorrow; scheduled (sent 1 day before) for later dates |

## Common errors

| Status | Meaning/Message | How to fix |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | A scheduling already exists (manual, automatic, canceled or with an error). Query it with `GET`; to reschedule, contact support |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Confirm that the current status is `shipped` and that the shipment belongs to the token user |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Take the package to a drop-off point |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Use a date from tomorrow onward |
| `400` | `Coleta não pode ser realizada em um final de semana` | Choose a business day |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Choose another business day |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Reduce the advance period |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | BR sender: fill in `number` and `district`. Other countries: check `postal_code` or send `state_code` |
| `400` | `O atributo federal_tax_id não é válido` | BR sender: inform a valid CPF or CNPJ |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Use a `state_code` of the sender country |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Check the `postal_code` or inform the `state_code` |
| `400` | `Agendamento não foi encontrado para esta remessa` | In the `POST`/`DELETE` with AWB: check the number, the age of the shipment (90 days) and whether it belongs to your account |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | Only cancel schedulings in `success` of shipments in `shipped` |
| `403` | `Has no permission!` | Ask support for the operation permission (create, query or cancel scheduling) |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contact support |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Check the `Authorization: Bearer ...` header |
| `404` | `Agendamento não foi encontrado para esta remessa` | In the `GET` with UUID: the shipment has no scheduling or does not belong to your account |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + field map in `data` | Fix the listed fields (e.g., `O campo data é obrigatório.`, `A data informada para o campo :attribute não respeita o formato :format.`, `O valor selecionado para o campo shipping é inválido.`, `O campo reason não contém um valor válido.`) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | `POST`: check whether all `address` keys were sent. `GET` with AWB: the AWB was not found (nonexistent, older than 90 days or from another account). In other cases, retry with backoff and contact support |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Unexpected failure in the `DELETE`. Retry with backoff; if it persists, contact support |

> Attention: in the **query by AWB** (`GET /v1/schedule/{awb}`), an AWB that is not found responds `500`, and not
> `404`. Prefer querying by the quotation UUID, which responds `404` when there is no scheduling.

## Final notes
- Always query the scheduling before creating or canceling: this avoids redundant requests and the error
  `já tem um agendamento em progresso`.
- Pay attention to the naming difference: **`initial_time` in the request** and **`inicial_time` in the response**.
- Scheduled pickups remain in `processing` until 1 day before the pickup. This is not an error. If the status
  does not change after the pickup date, contact support.
- Use `status.code` (and not `status.description`) for the logic of your integration.
- Shipment status: [Tracking](api-rastreio.md). Documents for the pickup: [Documents](api-documentos.md).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
