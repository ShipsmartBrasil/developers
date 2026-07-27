# Quote V2

## Overview
- Generates quotes with a simplified contract for new integrations.
- Use it when you only need to create the v2 quote.
- The detailed result lookup does not exist in this version in the analyzed source.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Language header: `lang: pt|en|es`.

## Conventions
- `Content-Type: application/json`
- `level` exists in the create payload and controls the quote output.

## Usage Flow
1. Create the quote in v2.
2. Receive the `quotation` and reuse it for the v2 shipment.
3. There is no quote lookup endpoint in v2 in the analyzed source.

## Endpoint
### Create quote
- Method and route: `POST /v2/quotation`
- Important rules: `object`, `type`, `tax`, `insurance`, `currency_quote`, `currency_payment`, `address_sender`, `address_receiver`, and `boxes` and/or `items` depending on the type.

### Main differences from v1
- v2 explicitly documents `items` as a quote type.
- v2 simplifies address fields in the public contract.
- The detailed result is returned in the `POST` response itself.
