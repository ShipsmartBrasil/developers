# Spedizione

- [Torna al README della lingua](README.md)

## Panoramica
- Crea, aggiorna, annulla e consulta le spedizioni.
- Espone anche documenti e tracking.
- È il flusso legacy più completo.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token in `Authorization`.
- Lingua opzionale: `lang`.

## Convenzioni
- `Content-Type: application/json`
- La spedizione usa `quotation` come collegamento alla quotazione.
- La quotazione deve appartenere alla stessa versione API.
- `payment.type` definisce il modello finanziario.

## Flusso d'uso
1. Crea la quotazione.
2. Verifica l'idoneità.
3. Invia la spedizione con il `quotation`.
4. Consulta `status` fino all'elaborazione.
5. Scarica i documenti in `/docs`.
6. Se necessario, programma il ritiro.

## Regole di business
- `package`/`pallet` con tipo `simple` non sono idonei.
- La quotazione non può essere scaduta.
- La spedizione non può già esistere per la stessa quotazione.
- Il tipo di pagamento deve essere abilitato per l'utente.
- Una volta creata, la spedizione è irreversibile nel normale funzionamento.
- Le modifiche successive sono ammesse solo in casi eccezionali.

## Endpoint
- `POST /v1/shipping`
- `GET /v1/shipping/{shipping}/status`
- `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`
- `GET /v1/shipping/{shipping}/docs`
- `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`
- `PUT /v1/shipping/{shipping}`
- `DELETE /v1/shipping/{shipping}`
