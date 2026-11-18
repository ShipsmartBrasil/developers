# Getting Started

- [Back to the language README](README.md)

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
| `404` | `{"status":"warning","message":"..."}` | Token not found/invalid |
| `403` | `{"status":"warning","message":"..."}` | Account blocked |
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

> The header is optional. If it is not sent (or sent with an unknown value), the API responds in Portuguese.

> **About the examples in this guide:** all response messages reproduced in the following pages
> use the API's default language (`pt`). By sending `lang: en` or `lang: es`, the `message` text comes
> translated, but the `status` and the HTTP code are always the same.

## Request conventions

| Item | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Encoding | UTF-8 |
| Dates (response) | `Y-m-d H:i:s` |
| Dates (scheduling) | `Y-m-d` for the date and `H:i:s` for the time |
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
generate a shipment in v2. Mixing versions returns `422` with the message
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
| `warning` | `400`, `403`, `404`, `406`, `422` | A business rule or validation blocked the operation. **The payload must be fixed** |
| `error` | `500` | Unexpected server-side failure. Try again; if it persists, contact support |

## Validation errors (`422`)

When a field fails validation, the response brings `data` with the map `field → list of messages`.

```json
{
  "status": "warning",
  "message": "Validation Error",
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
| `400` | A business rule blocked the operation (e.g. nonexistent payment type, expired quotation, no eligible carrier) | Read `message` and fix the scenario |
| `403` | No permission or blocked account | Ask support to grant access |
| `404` | Resource not found (token, shipment, contract not listed in the quotation) | Check the identifier you sent |
| `406` | Payload structure incompatible with what is expected | Review field types and nesting |
| `422` | Field validation failure | Fix the fields listed in `data` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Account presets

Some options can be **preconfigured in your account** in the panel. When configured, they are
applied automatically in the quotation and in the shipment, and you can omit the field in the payload.

| Preset | Affects | Note |
|---|---|---|
| Sales channel (`channel_code`) | Quotation | Used when you do not inform the channel |
| Tax (`tax`) | Quotation | `sender` or `receiver` |
| Insurance (`insurance`) | Quotation | - |
| Object (`object`) and type (`type`) | Quotation | - |
| Currencies (`currency_quote`, `currency_payment`) | Quotation | - |
| Measurement system (`measurement`) | Quotation | `metric` or `imperial` |
| Residential delivery / non-stackable | Quotation | - |
| Sender address | Quotation and shipment | Applied as `address_code` |
| Declared freight (`declared_freight`) | Quotation | Value + currency |
| Biller (`biller`) | Shipment | Address shown on the invoice |
| Box grouping | Quotation | `default`, `all_in_one` or `smartbox` |

**Precedence:** what you send in the payload always beats the preset. Exceptions:
- If you inform `address_sender.country_code` in the quotation, the sender address preset is discarded.
- If you inform `address_sender.street` in the shipment, the sender address preset is discarded.
- If you inform `declared_freight`, the declared freight preset is discarded.

### Box grouping
The grouping defines how the items are distributed across the boxes when you use `type: items`.
It is **not** a payload field: it is an account setting.

| Mode | Behavior |
|---|---|
| `default` | Platform default grouping |
| `all_in_one` | Consolidates everything into a single box |
| `smartbox` | Fits the items into the registered boxes considering volume and dimensions (with 10% slack) and respecting the item's `is_groupable` |

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
