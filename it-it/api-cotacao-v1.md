# Quotazione

- [Torna al README della lingua](README.md)

## Panoramica
- Genera quotazioni internazionali con dettaglio di trasporto, assicurazione e tasse.
- Usala per simulare la spedizione prima di creare la spedizione.
- La quotazione è la base per spedizione, tracking e documenti.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token in `Authorization`.
- Lingua accettata: `lang: pt|en|es`.
- La lingua predefinita è `pt`.

## Convenzioni
- `Content-Type: application/json`
- Date nel formato `Y-m-d H:i:s` nelle risposte.
- Valute in codice ISO 4217 a 3 lettere.
- `level` controlla il dettaglio del risultato: `simple` o `detailed`.

## Flusso d'uso
1. Invia una quotazione con origine, destinazione, oggetto, tipo e articoli/scatole.
2. Ricevi il `quotation` restituito dall'API.
3. Consulta il risultato con `GET /v1/quotation/{quotation}/{level}`.
4. Se idonea, usa lo stesso `quotation` per creare la spedizione.

## Funzionale
- `simple`: calcola per scatola con peso/valore diretto.
- `advanced`: calcola con scatole e articoli distribuiti per `box_id`.
- `items`: la validazione backend accetta questo tipo, ma il contratto OpenAPI non lo espone chiaramente.

## Idoneità per la spedizione
- La quotazione deve esistere, non essere scaduta, appartenere alla stessa versione API della spedizione e non essere già stata usata.
- `package`/`pallet` con tipo `simple` non sono idonei.
- La spedizione richiede anche un contratto compatibile e un tipo di pagamento consentito.
