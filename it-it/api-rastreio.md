# Tracking spedizione

- [Torna al README della lingua](README.md)

## Panoramica
- Consulta lo stato operativo e lo storico di una spedizione.

## Formati di risposta
- Lo stesso endpoint restituisce `timeline` e `status`.
- `timeline` elenca gli eventi nell'ordine di arrivo.
- `status` raggruppa gli eventi per stato.

## Endpoint
- `GET /v1/shipping/{shipping}/status`
- `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`
