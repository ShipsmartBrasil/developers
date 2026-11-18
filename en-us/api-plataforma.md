# Platform

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
- Utility endpoint that returns the name, the environment and the version of the Public API.
- It is the simplest and cheapest call of the API — ideal to **validate the token and connectivity** before
  any integration, and for a **health check** in monitoring.

## Base URL
- Production: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Interactive documentation: `https://api.shipsmart.global/doc` and `https://hmlapi.shipsmart.com.br/doc`

## Authentication
```
Authorization: Bearer YOUR_TOKEN
```

> This endpoint **requires a valid token**, but it does not require any specific permission.
> That is why it is the best test to find out whether your token is working.

## Endpoint: platform version

- **Method and route:** `GET /v1/platform/version`
- **Parameters:** none.

### Request example

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Accept: application/json'
```

### Response example

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
| `data.name` | string | Application name |
| `data.env` | string | Environment you are connected to (e.g. `production`, `staging`) |
| `data.version` | string | Published version of the API |

> Check the `env` to make sure you are not pointing to the wrong environment — it is a common mistake
> in integrations that share configuration between environments.

## Use cases

| Case | How to use it |
|---|---|
| Validate a new token | If it responds `200`, the token is valid |
| Diagnose an authentication `404` | If this endpoint fails too, the problem is the token, not the route |
| Confirm the environment | Read `data.env` |
| Monitoring health check | Lightweight call, with no side effects |
| Record the version in your logs | Useful to correlate behavior with API releases |

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `404` | Token not found or invalid | Check the `Authorization: Bearer ...` header and generate a new token in the panel |
| `403` | Account blocked | Contact support |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- This endpoint is **read-only** and has no effect at all on quotations, shipments or registries.
- Calls without the version prefix (`/platform/version`) are redirected with `308` to
  `/v1/platform/version`. Always prefer to state the version.
