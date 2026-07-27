# Spedizione V2

- [Torna al README della lingua](README.md)

## Panoramica
- Crea spedizioni con contratto semplificato.
- La fonte analizzata espone solo la creazione in v2.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token in `Authorization`.
- Lingua: `lang`.

## Convenzioni
- `Content-Type: application/json`
- `quotation` deve essere creato in v2.
- `payment.type` continua a definire il flusso finanziario.

## Flusso d'uso
1. Crea la quotazione in v2.
2. Invia la spedizione con il `quotation` restituito.
3. Continua il flusso con gli endpoint v1 quando necessario.

## Endpoint
- `POST /v2/shipping`

## Differenze rispetto a v1
- v2 lascia solo l'endpoint di creazione nella documentazione pubblica.
- `attachments` accetta solo PDF o XML nella fonte analizzata.
