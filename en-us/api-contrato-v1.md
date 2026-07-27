# Contract

- [Back to English README](README.md)

## Overview
- Lists the contracts available to the authenticated user.
- Contracts are used in quotes and shipments.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Optional language header: `lang`.

## Conventions
- `Content-Type: application/json`
- Listing can be filtered by `status`.

## Usage Flow
1. Authenticate the user.
2. List available contracts.
3. Use the contract `code` in the quote or shipment.

## Endpoint
### List contracts
- `GET /v1/contract`
