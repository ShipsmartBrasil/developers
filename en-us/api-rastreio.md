# Shipment Tracking

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

## Overview
- Queries the operational status and the full history of a shipment, from its receipt on the platform to the
  delivery (or the cancellation).
- The same response brings **two formats**: a chronological `timeline` and the events **grouped by status**.
- It works for shipments created in **v1 and v2**.

## Base URL
- Production: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Interactive documentation: `https://api.shipsmart.global/doc` and `https://hmlapi.shipsmart.com.br/doc`

## Authentication
```
Authorization: Bearer YOUR_TOKEN
lang: pt
```

> The status descriptions (`status_description`) are translated according to the `lang` header.
> The codes (`status_code`) do **not** change with the language — use them for integration logic.

## Endpoints

### Status by shipment
- **Method and route:** `GET /v1/shipping/{shipping}/status`

### Status by channel
- **Method and route:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

## Parameters

| Name | Location | Type | Required | Description |
|---|---|---|---|---|
| `shipping` | path | string | **yes** | Quotation UUID, AWB number (last 90 days) or NF-e key (44 digits) |
| `channel_identifier_number` | path | string | **yes** (by-channel variant) | Commercial identifier informed in the quotation |
| `channel_code` | path | integer | no | Sales channel code |

## Request example

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

By channel:
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/status/PED-2026-0001/1 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'lang: pt'
```

## Response example

```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Fedex Economy Startup Test BR",
      "tracking_number": "271249639538",
      "timeline": [
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:40",
          "status_code": "POT0001",
          "status_description": "Pagamento aprovado junto ao gateway: approved"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:44:38",
          "status_code": "SPD0001",
          "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
        },
        {
          "update_date": "2026-02-22",
          "update_time": "09:10:02",
          "status_code": "CLD0001",
          "status_description": "Remessa coletada pelo transportador"
        },
        {
          "update_date": "2026-02-23",
          "update_time": "04:35:11",
          "status_code": "ITR0001",
          "status_description": "Remessa em trânsito internacional"
        }
      ]
    },
    "cost_center": {
      "code": 1,
      "name": "E-commerce"
    },
    "received": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:38",
        "status_code": "RCV0001",
        "status_description": "Remessa foi recebida em nosso sistema"
      }
    ],
    "paid_out": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:40",
        "status_code": "POT0001",
        "status_description": "Pagamento aprovado junto ao gateway: approved"
      }
    ],
    "shipped": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:44:38",
        "status_code": "SPD0001",
        "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
      }
    ],
    "collected": [
      {
        "update_date": "2026-02-22",
        "update_time": "09:10:02",
        "status_code": "CLD0001",
        "status_description": "Remessa coletada pelo transportador"
      }
    ],
    "in_transit": [
      {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    ],
    "last_update": {
      "in_transit": {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    }
  }
}
```

## Response structure

| Field | Type | Description |
|---|---|---|
| `data.carrier.name` | string | Name of the contract/carrier that processed the shipment |
| `data.carrier.tracking_number` | string | Tracking number (AWB) at the carrier |
| `data.carrier.timeline` | array | **All events in chronological order.** Use this field to build a timeline in your interface |
| `data.cost_center` | object | **Optional.** Only appears when the shipment has a cost center |
| `data.cost_center.code` | integer | Cost center code |
| `data.cost_center.name` | string | Cost center name |
| `data.<status_group>` | array | Events grouped by status. The existing keys vary according to the progress |
| `data.last_update` | object | **Last event**, in the format `{ "<group>": { ...event } }`. Use it to know the current state with a single read |

Each event (in `timeline`, in the groups and in `last_update`) has the same shape:

| Field | Type | Description |
|---|---|---|
| `update_date` | string | Event date (`Y-m-d`) |
| `update_time` | string | Event time (`H:i:s`) |
| `status_code` | string | Internal status code (see table) or the raw code from the gateway/carrier |
| `status_description` | string | Readable description, translated by the `lang` header |

### The two formats, and when to use each

| Format | Where it is | Use it to |
|---|---|---|
| Chronological | `data.carrier.timeline` | Display the full history to the end user |
| Grouped by status | `data.received`, `data.shipped`, `data.in_transit`... | Check whether the shipment **has already passed** through a given stage (`if (isset(data.shipped))`) |
| Last state | `data.last_update` | Dashboards and synchronizations: one read answers "where is the shipment" |

## Status groups and codes

| Group (key in `data`) | `status_code` | Meaning |
|---|---|---|
| `received` | `RCV0001` | The shipment was received on the platform |
| `processing_payment` | `PPY0001` | Waiting for the payment confirmation at the gateway |
| `paid_out` | `POT0001` | Payment approved |
| `pending_consolidation` | `PCD0001` | Waiting for consolidation (contracts with consolidation) |
| `consolidating` | `CDT0001` | Being consolidated |
| `processing_shipment` | `PSH0001` | Payment settled, waiting for the processing at the carrier |
| `shipped` | `SPD0001` | **Processed at the carrier. Documents available** |
| `collecting` | `CLG0001` | Pickup in progress |
| `collected` | `CLD0001` | Collected by the carrier |
| `in_transit` | `ITR0001` | In transit |
| `delivering` | `DLG0001` | Out for delivery |
| `delivered` | `DLD0001` | Delivered |
| `error` | `ERR0001` | Processing error |
| `cancelling` | `CCG0001` | Cancellation requested, being processed |
| `cancelled` | `CCD0001` | Cancelled |
| `foc` | `FOC0001` | Finished by the carrier |
| `eol` | `EOL0001` | End of the tracking life cycle |

> **`status_code` may bring the raw code from the gateway or from the carrier** when it exists
> (e.g. `approved`, `charge.succeeded`, the carrier's own codes). The `XXX0001` codes are the platform's
> internal mapping, used when there is no external code. For business logic, prefer the
> **group key** (`shipped`, `delivered`, `cancelled`...) instead of the `status_code`.

## Typical state flow

```
received
   │
   ├── processing_payment ──► paid_out        (payment via gateway)
   │
   └── (billed payment: goes straight ahead)
                │
                ├── pending_consolidation ──► consolidating   (when there is consolidation)
                │
                ▼
         processing_shipment
                │
                ▼
             shipped          ◄── documents available from here on
                │
                ▼
    collecting ──► collected ──► in_transit ──► delivering ──► delivered
                │
                └── error | cancelling ──► cancelled | foc | eol
```

## Business rules

- **Every shipment must have a payment record** for the status to be assembled. If there is none,
  the response is `404` with `Nenhuma entrada de pagamento foi registrada para esta remessa!`.
- With `payment.type: billed`, the shipment does not go through `processing_payment`: it goes straight to
  `processing_shipment`/`shipped`.
- With `payment.type: gateway`, the shipment stays in `processing_payment` until the gateway approves; only then
  does it advance to `processing_shipment`.
- The group changes from `processing_shipment` to `shipped` at the moment the **documents are generated**.
  That is the signal to call [Shipment Documents](api-documentos.md).
- When the shipment is in `cancelling`, `cancelled` or `error`, the `last_update` uses the platform's own
  description instead of the carrier's raw text.
- Shipments that go **29 days or more** without a tracking event in an eligible status may be cancelled
  automatically by the platform.
- The search by AWB works for shipments created in the **last 90 days**.

## Polling best practices

| Situation | Recommended interval |
|---|---|
| Right after creating the shipment (waiting for `shipped`) | 10 to 30 seconds, with backoff |
| Shipment already in `shipped`/`in_transit` | 1 to 4 hours |
| Shipment in `delivered`, `cancelled`, `foc` or `eol` | Stop polling: these are final states |

1. Read `data.last_update` to decide whether something changed — it is the cheapest read.
2. Stop polling in the final states.
3. Use exponential backoff on `500`.
4. Treat `400`/`404` as a business condition: do not repeat the same request in a loop.

## Common errors

| Status | Meaning | How to fix it |
|---|---|---|
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Check `{shipping}` or the channel identifier |
| `403` | No permission to query the status | Ask support for the permission |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Wait for the payment record; if it persists, contact support |
| `422` | Non-numeric `channel_code` | Send an integer in `channel_code` |
| `500` | Unexpected error | Retry with backoff; if it persists, contact support |

## Final notes
- `data.carrier.timeline` is the most complete view; the groups are an operational summary to make
  programmatic reading easier.
- Store the `tracking_number` to offer the end customer the tracking on the carrier's website.
- To download the label and the invoice as soon as the shipment reaches `shipped`, see
  [Shipment Documents](api-documentos.md).
