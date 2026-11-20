# Getting Started

- [Back to the language README](README.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
This page gathers everything that is common to **all** endpoints of the Shipsmart Global Public API:
environments, authentication, language, response format, versioning and error handling.

Read this page before any other. The remaining pages of the guide assume you already know these
conventions and do not repeat them in detail.

## Environments

| Environment | Base URL | Interactive documentation |
|---|---|---|
| Production | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Staging | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

- Use **staging** to develop and test the integration. No shipment created there is actually dispatched.
- New features show up in staging first. If a field described here does not exist in production yet,
  check it at `https://hmlapi.shipsmart.com.br/doc`.
- Accessing the root (`/`) redirects to `/doc`.

## Authentication

Every call (except the documentation) requires a personal access token in the `Authorization` header,
using the **Bearer** scheme.

```
Authorization: Bearer YOUR_TOKEN
```

| Item | Value |
|---|---|
| Type | Bearer token |
| Header | `Authorization` |
| Where to get it | Shipsmart Global panel, in your account settings area |
| Expiration | The token does not expire over time; it is revoked if the account is blocked or the token is regenerated |

### Authentication errors

| Status | Response | Meaning |
|---|---|---|
| `404` | `{"status":"warning","message":"Nenhum usuário encontrado para o token especificado!"}` | Token missing, not found or invalid |
| `403` | `{"status":"warning","message":"Usuário especificado encontra-se bloqueado em nosso sistema!"}` | Account blocked |
| `403` | `{"status":"warning","message":"Has no permission!"}` | The token is valid, but your profile does not have permission for the action |

> Permissions are granular per action (quote, ship, cancel, list documents, schedule pickup, etc.).
> If you receive `Has no permission!`, ask support to grant the corresponding permission.

## Response language

Success, warning and error messages are translated according to the `lang` header.

| Header | Accepted values | Default |
|---|---|---|
| `lang` | `pt`, `en`, `es` | `pt` |

```
lang: en
```

> The header is optional. If it is not sent, the API responds in Portuguese. If it is sent with an
> unsupported value (e.g. `fr`), the messages come in English.

> **About the examples in this guide:** all response messages reproduced in the following pages
> use the API's default language (`pt`). By sending `lang: en` or `lang: es`, the `message` text comes
> translated, but the `status` and the HTTP code are always the same.

## Request conventions

| Item | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Encoding | UTF-8 |
| Dates (response) | `YYYY-MM-DD HH:MM:SS` |
| Dates (scheduling) | `YYYY-MM-DD` for the date and `HH:MM:SS` for the time |
| Currencies | 3-letter ISO 4217 code (e.g. `BRL`, `USD`, `EUR`) |
| Countries | 2-letter ISO 3166-1 alpha-2 code (e.g. `BR`, `US`, `PT`) |
| Decimal separator | Period (`.`) — never a comma |
| Weights | Kilograms when `measurement` is `metric`; pounds when it is `imperial` |
| Dimensions | Centimeters when `measurement` is `metric`; inches when it is `imperial` |
| Volume | m³ when `measurement` is `metric`; ft³ when it is `imperial` |

## Versioning

The version is part of the path: `/v1/...` and `/v2/...`.

- `POST /v1/quotation` → quotation in the first version
- `POST /v2/quotation` → quotation in the second version

### Calls without a version prefix
If you call a resource **without** the version prefix, the API responds `308 Permanent Redirect`
pointing to the same resource under `/v1`, preserving the query string.

```
GET /quotation  →  308  →  GET /v1/quotation
```

> For new integrations, always state the version explicitly. Do not rely on the redirect.

### Compatibility between versions
A quotation created in v1 **can only** generate a shipment in v1, and a quotation created in v2 **can only**
generate a shipment in v2. Mixing versions returns `422` with the message (in `data.quotation`)
`Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.`

See [Quotation & Shipment: v1 vs v2](api-comparativo-cotacao-remessa.md) to choose the version.

## Default response format

All responses follow the same envelope:

```json
{
  "status": "success",
  "message": "Human-readable message",
  "data": {}
}
```

| Field | Type | Description |
|---|---|---|
| `status` | string | `success`, `warning` or `error` |
| `message` | string | Message already translated according to the `lang` header |
| `data` | object/array | Response content. Absent in responses that only confirm receipt |

### Relationship between `status` and HTTP

| `status` | Typical HTTP | How to read it |
|---|---|---|
| `success` | `200` | The operation was accepted/completed |
| `warning` | `400`, `401`, `403`, `404`, `406`, `422` | A business rule or validation blocked the operation. **The payload must be fixed** |
| `error` | `500` (and some weight/carrier refusals with `400`/`404`) | Unexpected failure or carrier refusal. Read `message`; `500` errors can be retried |

## Validation errors (`422`)

When a field fails validation, the response brings `data` with the map `field → list of messages`.
The top-level `message` is **generic** (it does not describe the error): always use the content of `data`.

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "object": ["O campo object é obrigatório."],
    "address_sender.country_code": [
      "O campo address sender.country code é obrigatório quando address sender.address code não está presente."
    ],
    "boxes": ["O campo boxes é obrigatório a menos que type esteja presente em items."]
  }
}
```

> For nested fields, the key uses a dot (`address_sender.country_code`) and, in arrays,
> it uses the index (`items.0.hscode`). Use this to point out the error in your form/integration.

## General HTTP status table

| Status | When it happens | What to do |
|---|---|---|
| `200` | Request accepted/processed | Continue the flow |
| `308` | Call without a version prefix | Use `/v1/...` or `/v2/...` |
| `400` | A business rule blocked the operation (e.g. invalid tax document, box without items, invalid ZIP code) | Read `message` and fix the scenario |
| `401` | Payment type not enabled for the account | Enable the payment option or use another type |
| `403` | No permission, blocked account, simple or expired quotation in the shipment | Ask support to grant access / redo the quotation |
| `404` | Resource not found (token, quotation from another account, shipment, contract not listed in the quotation, address) | Check the identifier you sent |
| `406` | Business conflict (quotation already used, incompatible registered address, cargo that requires personalized service) | Read `message` and adjust the flow |
| `422` | Field validation failure, or a business rule about the data (e.g. HS Code without a tax table, DDP-only account) | Fix the fields listed in `data` / read `message` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Account presets

Some options can be **preconfigured in your account** in the panel. They are applied by the API after
payload validation, with different rules per field:

| Preset | Affects | How it is applied |
|---|---|---|
| Tax (`tax`) | Quotation, Manifest, Draft | **Replaces** the `tax` sent. Destination countries configured as exceptions receive the opposite of the default. If the account is **locked to DDP**, the tax is always `sender` |
| Automatic insurance | Quotation, Manifest | Purchases insurance when the declared value (converted to USD) reaches the configured minimum, even with `insurance: false` |
| Sales channel (`channel_code`) | Quotation, Manifest, Draft | Used when the field is not sent |
| Measurement system (`measurement`) | Quotation, Manifest, Draft | Used when the field is not sent; if it is not configured either, `metric` |
| Residential delivery / non-stackable | Quotation, Draft | Used when the fields are not sent |
| Declared freight (`declared_freight`) | Quotation, Manifest, Draft | Used when the field is not sent. Requires the declared freight permission |
| Box grouping | Quotation v2 (`type: items`) | Defines how the boxes are assembled (see below). It is not a payload field |
| Cost center (`cost_center_code`) | Shipment v1/v2, Manifest, Draft | Used when the field is not sent **or is sent empty/null** |
| Shipment purpose (shipment `type`) | Shipment v1/v2, Draft | Used when the field is not sent |
| Biller (`biller`) | Shipment v1/v2, Manifest, Draft | Used when the field is not sent (or comes empty) and the account is configured to display a biller |
| Billing notification (`payment_notify_shipping_billing`) | Shipment v1/v2 | Used when the field is not sent |
| Automatic pickup | Shipment | Creates the pickup scheduling automatically (see [Pickup Scheduling](api-agendamento.md)) |

**Precedence:** except for the **tax** (which always follows the preset, when configured), what you send
in the payload beats the preset. Required payload fields (`object`, `type`, `insurance`, currencies,
`address_sender`) must always be sent — the preset does not exempt them.

> The sender address preconfigured in the panel does **not** replace the API's `address_sender`.
> To reuse a registered address, send `address_sender.address_code`.

### Box grouping
The grouping defines how the items are distributed across the boxes when you use `type: items`
(quotation by items, **exclusive to v2**).
It is **not** a payload field: it is an account setting.

| Mode | Behavior |
|---|---|
| `default` | Each item uses the box linked to its registration (`box_code`/`box_identifier`); groupable items with the same box are combined and non-groupable items receive one box per unit |
| `all_in_one` | Consolidates the groupable items into a single virtual box with the total volume (based on the registered dimensions); non-groupable items receive one box per unit |
| `smartbox` | Fits the items into the boxes **sent in `boxes[]`** (required), considering volume and dimensions with 10% slack and respecting the item's `is_groupable`/`is_foldable` |

Examples of each mode in [Items Quotation](api-cotacao-v2.md#items-quotation).

## Registrations made in the panel

Some codes used in the payloads come from registrations maintained **in the Shipsmart Global panel** (there is no public
endpoint to create them). They must exist, be active and belong to your account (or to the authorized accounts):

| Registration | Payload field | Where it is used |
|---|---|---|
| Address | `address_sender.address_code`, `address_receiver.address_code`, `manufacturer.address_code`, `biller.address_code` | Quotation, Shipment, Manifest, Draft |
| Sales channel | `channel_code` | Quotation, Manifest, Draft, queries and cancellation by channel |
| Cost center | `cost_center_code` | Shipment (creation and update), Manifest, Draft |
| Kit | `boxes[].kit` | Quotation, Manifest, Draft |

Boxes and items can also be registered through the API: see [Box](api-caixa-v1.md) and [Items](api-itens-v1.md).

## Accepted currencies

Same list for `currency_quote` (quotation currency) and `currency_payment` (payment currency):

`USD`, `CAD`, `NZD`, `TWD`, `HKD`, `AUD`, `SGD`, `BRL`, `MXN`, `ARS`, `CLP`, `COP`, `UYU`, `BOB`,
`PEN`, `PYG`, `EUR`, `CNY`, `GBP`, `JPY`, `CHF`, `THB`, `DKK`, `NOK`, `SEK`, `AED`, `TRY`, `ILS`,
`ZAR`, `SAR`, `RUB`, `INR`, `PLN`

> `currency_payment` with payment via gateway has availability that depends on the provider:
> `BRL` is processed by one provider and the other currencies by another. If the chosen currency is not
> enabled for your account, the shipment returns `warning` at payment time.

## First call (connectivity test)

The simplest endpoint to validate the token and connectivity:

```bash
curl --request GET \
  --url https://hmlapi.shipsmart.com.br/v1/platform/version \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Accept: application/json'
```

Response:

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "name": "Shipsmart Global API",
    "env": "staging",
    "version": "1.0.0"
  }
}
```

## Integration best practices

1. **Always store the `quotation`** returned by the quotation: it is the key to the whole rest of the flow
   (shipment, status, documents, scheduling).
2. **Also store your commercial identifier** in `channel_identifier_number`. This allows you to query
   status, documents and to cancel using your order number, without depending on the `quotation`.
3. **Never poll aggressively.** Status/document queries must have an interval of a few
   seconds and exponential backoff.
4. **Treat `warning` as a business error** (do not repeat the same request) and `error` as transient
   (you may repeat it).
5. **Respect the field size limits** described on each page. Longer texts are rejected
   with `422`, not truncated.
6. **Quotations expire.** Create the shipment within the validity informed in `valid_until` of each carrier.

## Next steps

- [Quotation](api-cotacao-v1.md) or [Quotation V2](api-cotacao-v2.md)
- [Shipment](api-remessa-v1.md) or [Shipment V2](api-remessa-v2.md)
- [Quotation & Shipment: v1 vs v2](api-comparativo-cotacao-remessa.md)
- [Error Codes](api-codigos-de-erro.md)
