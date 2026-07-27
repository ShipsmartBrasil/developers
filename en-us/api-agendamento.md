# Pickup Scheduling

- [Back to English README](README.md)

## Overview
- Creates, queries, and cancels pickup schedules linked to a shipment.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Language header: `lang`.

## Conventions
- Dates use `Y-m-d`.
- Times use `H:i:s`.
- The schedule must respect the sender's country.

## Usage Flow
1. Ensure the shipment is ready for pickup scheduling.
2. Create the pickup window and instruction.
3. Check the schedule status.
4. Cancel it if needed using an allowed reason.

## Endpoints
### Create schedule
- `POST /v1/schedule`

### Query schedule
- `GET /v1/schedule/{shipping}`

### Cancel schedule
- `DELETE /v1/schedule/{shipping}/{reason}`
