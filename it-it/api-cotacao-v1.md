# Quotazione

## Panoramica
- Genera quotazioni internazionali con dettaglio di trasporto, assicurazione e tasse.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token in `Authorization`.

## Flusso d'uso
1. Invia la quotazione con origine, destinazione, tipo oggetto e colli/articoli.
2. Ricevi il codice `quotation`.
3. Consulta il risultato con `GET /v1/quotation/{quotation}/{level}`.
4. Se idonea, riutilizza la stessa quotazione per creare la spedizione.
