# Quotazione V2

- [Torna al README della lingua](README.md)

## Panoramica
- Genera quotazioni con contratto semplificato per nuove integrazioni.
- Usala quando serve solo creare la quotazione v2.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token in `Authorization`.

## Endpoint
- `POST /v2/quotation`

## Differenze principali rispetto a v1
- v2 documenta esplicitamente `items`.
- v2 semplifica i campi indirizzo.
- Il risultato dettagliato ritorna direttamente nella risposta del `POST`.
