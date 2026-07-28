# Pickup Scheduling

- [Back to language README](README.md)

## Overview
- Creates, retrieves, and cancels pickup schedules linked to the shipment.
- Pickup can be created by quotation/shipment or by AWB.
- Associated with the main operational flow.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Language header: `lang`.

## Conventions
- Dates in `Y-m-d` format.
- Times in `H:i:s` format.
- The schedule must respect the sender's country.

## Usage Flow
1. Ensure the shipment is ready for scheduling.
2. Create the pickup with a time window and instruction.
3. Check the schedule status.
4. If necessary, cancel using one of the allowed reasons.

## Endpoints
### Create schedule
- Method and route: `POST /v1/schedule`
- Important rules: provide `shipping` or `awb_number`.

### Retrieve schedule
- Method and route: `GET /v1/schedule/{shipping}`

### Cancel schedule
- Method and route: `DELETE /v1/schedule/{shipping}/{reason}`

### Parameters
| Name | Location | Type | Required | Description | Example | Notes |
|---|---|---:|---:|---|---|---|
| `shipping` | body/path | string | conditional | Quotation/shipment UUID | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Required if `awb_number` is not provided |
| `awb_number` | body | string | conditional | AWB | `271249639538` | Required if `shipping` is not provided |
| `date` | body | date | yes | Pickup date | `2023-11-21` | Format `Y-m-d` |
| `initial_time` | body | time | yes | Window start | `08:00:00` | Format `H:i:s` |
| `final_time` | body | time | yes | Window end | `18:00:00` | Format `H:i:s` |
| `instruction` | body | string | yes | Pickup instruction | `Deixar na portaria` | 5 to 80 characters |
| `address` | body | object | no | Alternate address | - | Must be in the same country as the sender |
| `reason` | path | string | yes | Cancellation reason | `001` | Fixed code |

### Cancellation reasons
| Code | Meaning |
|---|---|
| `001` | Package not ready |
| `002` | Rates too high |
| `003` | Transit time too slow |
| `004` | Take to service center or drop box |
| `005` | Commitment time not met |
| `006` | Reason not given |
| `007` | Other |
| `008` | Pickup modified |

### Request example
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2023-11-21",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Deixar na portaria"
  }'
```

### Response example
```json
{
  "status": "success",
  "message": "Agendamento de coleta registrado com sucesso"
}
```

## Business Rules
- Scheduling is only possible if the shipment is ready for it.
- There may already be a schedule in progress, in which case the API returns a warning.
- The alternate address must respect the same country as the sender.

## Common Errors
| Status | Meaning | How to Fix |
|---|---|---|
| `400` | Schedule already exists or shipment is not ready | Adjust the flow | 
| `422` | Validation | Correct date, time, or required fields |
| `500` | Unexpected error | Contact the responsible team |

## Final Notes
- Check the status before canceling to avoid redundant requests.
