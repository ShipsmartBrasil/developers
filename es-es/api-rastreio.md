# Rastreo de remesa

- [Volver al README del idioma](README.md)

## Visión general
- Consulta el estado operativo y el historial de una remesa.

## Formatos de respuesta
- El mismo endpoint devuelve `timeline` y `status`.
- `timeline` lista los eventos en orden de llegada.
- `status` agrupa los eventos por estado.

## Endpoints
- `GET /v1/shipping/{shipping}/status`
- `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`
