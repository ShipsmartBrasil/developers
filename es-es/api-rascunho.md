# Borrador

- [Volver al README del idioma](README.md)

## Visión general
- Crea, lista, actualiza y elimina borradores de envío.

## Convenciones
- Acepta payloads parciales.
- El status puede ser `received`, `edited`, `validating`, `incomplete`, `complete`, `processing` o `error`.

## Endpoints
- `POST /v1/draft`
- `PUT /v1/draft/{code}`
- `GET /v1/draft`
- `DELETE /v1/draft`
