# Platform

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Utility endpoint that returns the name, the environment and the version of the Public API.
- It is the simplest and cheapest call of the API — ideal to **validate the token and connectivity** before
  any integration, and for a **health check** in monitoring.
- It only requires a valid token: **it does not depend on any permission** of the profile.
- It is read-only and has no side effects.
- This endpoint does not appear in the interactive documentation (`/doc`); this page is its reference.

## Base URL
- Production: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Interactive documentation: `https://api.shipsmart.global/doc` and `https://hmlapi.shipsmart.com.br/doc`

## Authentication
```
Authorization: Bearer YOUR_TOKEN
lang: pt
```

> This endpoint **requires a valid token**, but it does not require any specific permission.
> That is why it is the best test to find out whether your token is working.
> The `lang` header (`pt`, `en`, `es`) only changes the text of `message`.

## Usage flow

```
1. GET /v1/platform/version   → 200: valid token and API reachable
2. Check data.env             → expected environment (production or staging)
3. Move on to the integration → quotation, shipment etc.
```

## Endpoint: platform version

- **Method and route:** `GET /v1/platform/version`
- **When to use:** when configuring a new token, when switching environments and in periodic health checks.

### Parameters

No path, query or body parameters. Only the [Authentication](#authentication) headers.

### Minimum payload

No body:

```
GET /v1/platform/version
```

### Maximum payload

There are no optional parameters. The only possible variation is the language of the message, via header:

```
GET /v1/platform/version
lang: en
```

### Request example

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Accept: application/json' \
  --header 'lang: pt'
```

### Response example

`200 OK`

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "name": "Shipsmart Global API",
    "env": "production",
    "version": "1.0.0"
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | Always `success` when the token is accepted |
| `message` | string | `Registro(s) listado(s) com sucesso.` (translated according to the `lang` header) |
| `data.name` | string \| null | Application name |
| `data.env` | string \| null | Environment you are connected to (e.g. `production`, `staging`) |
| `data.version` | string \| null | Published version of the API |

> Check the `env` to make sure you are not pointing to the wrong environment — it is a common mistake
> in integrations that share configuration between environments.
>
> The values in `data` are informational and defined by each environment's configuration. Do not use them as
> a logic key (e.g. exact version comparison) and treat `null` as a possible value.

## Use cases

| Case | How to use it |
|---|---|
| Validate a new token | If it responds `200`, the token is valid |
| Diagnose an authentication `404` | If this endpoint also responds `404`, the problem is the token, not the route |
| Diagnose `403 Has no permission!` on another route | If this endpoint responds `200`, the token is valid and only the permission for that operation is missing |
| Confirm the environment | Read `data.env` |
| Monitoring health check | Lightweight call, with no side effects |
| Record the version in your logs | Useful to correlate behavior with API releases |

## Business rules

| Rule | Detail |
|---|---|
| Authentication | The token is validated before any response. Without the `Authorization` header, the response is `404` |
| Permission | None. Any valid token of an active user receives `200` |
| Blocked account | A token of a blocked user receives `403` |
| Version in the path | Calls without the prefix (`/platform/version`) receive `308` to `/v1/platform/version`, preserving the query string |
| Read-only | Does not change quotations, shipments or registrations |

## Common errors

| Status | Meaning/Message | How to fix it |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Token missing, incorrect or regenerated. Check the `Authorization: Bearer ...` header and, if needed, generate a new token in the panel |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Blocked account. Contact support |
| `400` | `status: "error"` with the description of the authentication failure | Unexpected failure while validating the token. Retry with backoff; if it persists, contact support |
| `308` | Permanent redirect | Call without `/v1`. Use the full route `/v1/platform/version` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- This endpoint is **read-only** and has no effect at all on quotations, shipments or registrations.
- Calls without the version prefix (`/platform/version`) are redirected with `308` to
  `/v1/platform/version`. Always prefer to state the version.
- Token, language and response format details: [Getting Started](api-primeiros-passos.md).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
