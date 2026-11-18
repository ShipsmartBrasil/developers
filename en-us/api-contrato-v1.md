# Contract

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
- Lists the shipping contracts available to the authenticated user.
- A "contract" is the combination of **carrier + service + commercial conditions** of your account.
  It is what the quotation returns in `carriers[]`.
- Use this endpoint for diagnostics and to know in advance which carriers can appear
  in your quotations.

> You **do not need** to call this endpoint in the normal integration flow. The quotation already returns the
> eligible contracts for that specific shipment, with price and transit time.

## Base URL
- Production: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Interactive documentation: `https://api.shipsmart.global/doc` and `https://hmlapi.shipsmart.com.br/doc`

## Authentication
```
Authorization: Bearer YOUR_TOKEN
lang: pt
```

## When to use it

| Situation | How it helps |
|---|---|
| The quotation returns `Nenhum transportador encontrado para este usuário!` | Confirms whether the account has any active contract |
| The quotation returns `Nenhum contrato disponível para este usuário!` | Confirms the contract assignment |
| You want to show the enabled carriers to the user | Lists names and validity periods |
| You received `Contrato especificado não foi listado para esta cotação!` | Helps you check the expected `code`/`slug` |

## Endpoint: list contracts

- **Method and route:** `GET /v1/contract`

| Parameter | Location | Type | Required | Default | Values | Description |
|---|---|---|---|---|---|---|
| `status` | query | string | no | `active` | `active`, `inactive`, `all` | Filters by the contract validity. Unknown values fall back to the `active` default |
| `page` | query | integer | no | `1` | - | Listing page |

- **Items per page:** 15

### Request example

```bash
# Active contracts (default)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'

# All contracts, active and inactive
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

### Response example

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "code": 3,
        "contract_description": "Standard PRO contract",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "DHL",
        "initial_date": "2026-01-06 00:00:00",
        "final_date": "2026-12-31 23:59:59",
        "active": "active",
        "direct_quotation": false
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `code` | integer | **Contract code.** It is the same value that appears in `carriers[].code` in the quotation and that you send in `carrier_code` in the shipment |
| `contract_description` | string | Commercial description of the contract |
| `contract_name` | string | Contract name |
| `country` | string | Country of origin of the contract (ISO 3166-1 alpha-2) |
| `carrier` | string | Carrier name |
| `initial_date` | string | Start of the validity (`Y-m-d H:i:s`) |
| `final_date` | string | End of the validity (`Y-m-d H:i:s`) |
| `active` | string | `active` or `inactive` |
| `direct_quotation` | boolean | `true` when the contract quotes directly at the carrier (in real time), instead of using a freight table |

## Relationship with the quotation and the shipment

```
GET /v1/contract            → contracts enabled in the account (overview)
        │
POST /v1/quotation          → carriers[] : contracts eligible FOR THAT SHIPMENT
        │                     (with code, slug, price, transit time and valid_until)
        ▼
POST /v1/shipping           → chosen carrier_code (or slug)
```

| Point of attention | Explanation |
|---|---|
| The quotation may return **fewer** contracts than this endpoint | The contract exists, but it does not serve that weight/dimension/route |
| The shipment accepts **only** contracts listed in the quotation | Sending an active `carrier_code` that is absent from the quotation returns `404 Contrato especificado não foi listado para esta cotação!` |
| The `slug` does not appear in this listing | Use the `slug` returned in `carriers[]` in the quotation |

## Business rules
- The return depends on **your user's assignment** to the contracts.
- If no contract is found with the informed filter, the platform performs a second search
  considering the account's default contracts.
- `inactive` contracts (outside the validity period) do **not** appear in the quotations.
- `direct_quotation: true` indicates real-time quotation at the carrier. These contracts may take
  a bit longer to respond and depend on valid credentials
  (`Credenciais inválidas para cotação direta`).

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `403` | No permission to list contracts | Ask support for the permission |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- This endpoint is **read-only**: contracts are configured commercially, not via the API.
- Use it as a diagnostic tool when the quotation does not return any carrier.
- To understand why a specific contract did not appear in the quotation, see
  **Cargo restrictions** in [Quotation (v1)](api-cotacao-v1.md#cargo-restrictions).
