# Manifest

- [Back to English README](README.md)

## Overview
- Executes the synchronous quote + shipment flow.
- May return the final operation documents after creation.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in `Authorization`.
- Language header: `lang`.

## Conventions
- Optional `format` query: `pdf` or `zpl`.
- The published contract uses `billed` as the payment type.

## Usage Flow
1. Send the complete manifest payload.
2. The API quotes, creates the shipment, and generates documents.
3. If `format` is `pdf` or `zpl`, the response favors that format.

## Endpoint
### Create manifest
- `POST /v1/manifest`
