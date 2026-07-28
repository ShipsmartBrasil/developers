# Contract

- [Back to language README](README.md)

## Overview
- Lists the contracts available to the authenticated user.
- Contracts are used in quotation and shipment creation.
## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Optional language header: `lang`.

## Conventions
- `Content-Type: application/json`
- The listing can filter by `status`.

## Usage Flow
1. Authenticate the user.
2. List the available contracts.
3. Use the contract `code` in the quotation or shipment.

## Endpoints
### List contracts
- Method and route: `GET /v1/contract`
- When to use: to identify active/inactive contracts before quotation.

## Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `status` | query | string | no | Filters contracts | `active` | `active`, `inactive`, `all` |

## Minimum and Maximum Payload
- Does not apply to the listing endpoint.

## Request example
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=active' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Response example
```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "code": 3,
        "contract_description": "Contrato padrão PRO",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "Carrier Name",
        "initial_date": "2023-03-06 00:00:00",
        "final_date": "2023-06-30 23:59:59",
        "active": "active",
        "direct_quotation": false
      }
    ]
  }
}
```

## Business Rules
- If there are no contracts in the first filter, the API attempts a second search with alternative backend behavior.
- The response depends on the user's association with the contracts.

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `500` | Unexpected error | Contact the responsible team |
