# Contract

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Lists the shipping contracts available to the authenticated user, in a paginated way.
- A "contract" is the combination of **carrier + service + commercial conditions** of your account.
  It is what the quotation returns in `carriers[]`.
- Use this endpoint for diagnostics and to know in advance which carriers can appear
  in your quotations.
- Read-only: contracts are configured commercially, not via the API.

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

> Your profile must have the permission to list contracts. Without it, the response is `403` with `Has no permission!`.

## Usage flow

```
1. GET /v1/contract                 → active and currently valid contracts (default)
2. GET /v1/contract?status=all      → also include the deactivated ones
3. GET /v1/contract?status=all&page=2 → next pages (repeat the status on each page)
```

### When to use it

| Situation | How it helps |
|---|---|
| The quotation returns `Nenhum transportador encontrado para este usuário!` | Confirms whether the account has any active and currently valid contract |
| The quotation returns `Nenhum contrato disponível para este usuário!` | Confirms the contract assignment |
| You want to show the enabled carriers to the user | Lists names and validity periods |
| You received `Contrato especificado não foi listado para esta cotação!` | Helps you check the expected `code` |

## Key concepts

| Concept | Meaning |
|---|---|
| Assigned contract | Contract associated directly with your account. If your user is linked to a main account, the main account's contracts apply |
| Default contract | Default platform contract, used when the account has no assigned contracts |
| Validity | Period between `initial_date` and `final_date`. **Only contracts valid today are listed**, regardless of the `status` filter |
| Active / inactive | `active` = enabled contract; `inactive` = deactivated contract (still within the validity period). It does **not** refer to the validity |

## Endpoint: list contracts

- **Method and route:** `GET /v1/contract`
- **When to use:** diagnosing a quotation without carriers or displaying the enabled carriers.

### Parameters

| Parameter | Location | Type | Required | Default | Values | Description |
|---|---|---|---|---|---|---|
| `status` | query | string | no | `active` | `active`, `inactive`, `all` | Filters by contract status (enabled/deactivated). Empty or unknown values fall back to the `active` default, without error |
| `page` | query | integer | no | `1` | integer ≥ 1 | Listing page. Invalid values are treated as `1` |

- **Items per page:** 15 (fixed; there is no parameter to change it).
- **Ordering:** not guaranteed. Do not depend on the order of the items.

### Minimum payload

No body and no query string:

```
GET /v1/contract
```

### Maximum payload

All accepted parameters:

```
GET /v1/contract?status=all&page=2
```

### Request example

```bash
# Active contracts (default)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'

# All contracts, active and inactive, page 2
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all&page=2' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

### Response example

`200 OK`

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
        "direct_quotation": false,
        "active": "active"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/contract?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Response fields

Items of `data.data[]`:

| Field | Type | Description |
|---|---|---|
| `code` | integer | **Contract code.** It is the same value that appears in `carriers[].code` in the quotation and that you send in `carrier_code` in the shipment |
| `contract_description` | string \| null | Commercial description of the contract |
| `contract_name` | string | Contract name |
| `country` | string | Country of origin of the contract (ISO 3166-1 alpha-2) |
| `carrier` | string \| null | Name of the carrier service |
| `initial_date` | string | Start of the validity (`YYYY-MM-DD HH:MM:SS`) |
| `final_date` | string | End of the validity (`YYYY-MM-DD HH:MM:SS`) |
| `direct_quotation` | boolean | `true` when the contract quotes directly at the carrier (in real time), instead of using a freight table |
| `active` | string | `active` (enabled) or `inactive` (deactivated) |

Pagination (in `data`):

| Field | Type | Description |
|---|---|---|
| `current_page` | integer | Current page |
| `from` / `to` | integer \| null | Position of the first and last item of the page (`null` when the page is empty) |
| `total` | integer | Total number of contracts found |
| `per_page` | integer | Items per page (`15`) |
| `last_page` | integer | Number of the last page |
| `first_page_url` / `last_page_url` | string | URLs of the first and last page |
| `prev_page_url` / `next_page_url` | string \| null | URLs of the previous and next page (`null` when they do not exist) |
| `path` | string | Base URL of the listing |
| `links` | array | Navigation links (`url`, `label`, `active`). The `label` is translated according to the `lang` header |

> ⚠️ The pagination URLs **do not preserve the `status` filter**. When navigating with `status=inactive` or
> `status=all`, build the next page URL yourself (`?status=all&page=N`) instead of following
> `next_page_url` directly.

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
| The quotation may return **fewer** contracts than this endpoint | The quotation only considers active contracts of the shipment's country of origin, with a freight table for the object type (document or non-document) and that serve that weight/dimension/route |
| The shipment accepts **only** contracts listed in the quotation | Sending an active `carrier_code` that is absent from the quotation returns `404 Contrato especificado não foi listado para esta cotação!` |
| The `slug` does not appear in this listing | Use the `slug` returned in `carriers[]` in the quotation |

## Business rules

| Rule | Detail |
|---|---|
| Assignment | The return depends on the contracts assigned to your account (or to the main account, for linked users) |
| Default contracts | If the search does not find any assigned contract **on the requested page**, the platform returns the platform's default contracts with the same filter. That is why requesting a page beyond the last one may return default contracts |
| Validity | Contracts outside the validity period (before `initial_date` or after `final_date`) are never listed |
| `status` filter | `active` (default), `inactive` or `all`. Any other value is treated as `active` |
| Quotation | `inactive` (deactivated) contracts do **not** appear in the quotations |
| Direct quotation | `direct_quotation: true` indicates real-time quotation at the carrier. These contracts may take a bit longer to respond and depend on valid credentials (`Credenciais inválidas para cotação direta`) |

## Common errors

| Status | Meaning/Message | How to fix it |
|---|---|---|
| `403` | `Has no permission!` | Ask support for the permission to list contracts |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contact support |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Check the `Authorization: Bearer ...` header |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Retry with backoff; if it persists, contact support |

> Invalid filters do **not** generate errors: an unknown `status` becomes `active` and an invalid `page` becomes `1`.
> An empty list (`data.data: []`, `total: 0`) means that there are no currently valid contracts or default contracts
> for that filter.

## Final notes
- This endpoint is **read-only**: contracts are configured commercially, not via the API.
- Use it as a diagnostic tool when the quotation does not return any carrier.
- To understand why a specific contract did not appear in the quotation, see
  **Cargo restrictions** in [Quotation (v1)](api-cotacao-v1.md#cargo-restrictions).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
