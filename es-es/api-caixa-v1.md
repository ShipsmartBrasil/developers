# Caja

- [Volver al README del idioma](README.md)

## Visión general
- Gestiona el alta y consulta de cajas usadas en cotizaciones, borradores y remesas.

## Convenciones
- `Content-Type: application/json`
- La consulta usa `boxes[box_ids][]`.
- Alta/actualización es asíncrona.

## Endpoints
- `GET /v1/box`
- `POST /v1/box`
- `DELETE /v1/box`
