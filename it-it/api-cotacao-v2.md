# Quotazione V2

- [Torna al README della lingua](README.md)

## Panoramica
- Genera quotazioni con contratto semplificato per nuove integrazioni.
- Usa quando vuoi solo creare la quotazione nella v2.
- La consultazione dettagliata del risultato non esiste in questa versione nella fonte analizzata.
- Versione documentata: v2.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header della lingua: `lang: pt|en|es`.

## Convenzioni
- `Content-Type: application/json`
- `level` esiste nel request body/query della creazione e controlla il ritorno del calcolo.
- HTTP più comuni: `200`, `400`, `422`, `500`.

## Flusso di utilizzo
1. Invia la quotazione nella v2.
2. Ricevi il `quotation` e usa questo codice per creare la spedizione nella v2.
3. Non c'è endpoint di consultazione quotazione nella v2 nella fonte analizzata.

## Endpoints
### Creare quotazione
- Metodo e rotta: `POST /v2/quotation`
- Quando usare: nuovo flusso di pricing.
- Regole importanti: `object`, `type`, `tax`, `insurance`, `currency_quote`, `currency_payment`, `address_sender`, `address_receiver`, `boxes` e/o `items` secondo il tipo.

#### Differenze funzionali rispetto alla v1
- La v2 documenta `items` esplicitamente come tipo di quotazione.
- La v2 semplifica i campi di indirizzo nel contratto OpenAPI, ma la validazione continua a richiedere i dati necessari secondo origine/destinazione.
- Il risultato dettagliato viene consegnato nello stesso `POST`.

#### Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `object` | body | string | sì | Oggetto inviato | `package` | `doc`, `package`, `pallet`, `not_doc` nella fonte |
| `type` | body | string | sì | Tipo della quotazione | `items` | `simple`, `advanced`, `items` |
| `tax` | body | string | sì | Imposta | `sender` | `sender` o `receiver` |
| `insurance` | body | boolean | sì | Assicurazione | `true` | - |
| `currency_quote` | body | string | sì | Valuta di quotazione | `USD` | ISO 4217 |
| `currency_payment` | body | string | sì | Valuta di pagamento | `EUR` | ISO 4217 |
| `address_sender` | body | object | sì | Mittente | - | `address_code` o manuale |
| `address_receiver` | body | object | sì | Destinatario | - | `address_code` o manuale |
| `boxes` | body | array | sì | Scatole/pallet | - | Obbligatorio salvo quando `type=items` |
| `items` | body | array | sì/condizionale | Articoli | - | Regole dipendono dal tipo |
| `manufacturer` | body | object | no | Produttore | - | Opzionale |
| `declared_freight` | body | object | no | Nolo dichiarato | - | Opzionale |

#### Payload minimo
```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ]
}
```

#### Payload massimo
```json
{
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
  "residential_delivery": false,
  "non_stackable": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 8, "width": 8, "depth": 8 },
    { "box_code": 3, "height": 10, "width": 10, "depth": 10 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "US",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100
    }
  ],
  "manufacturer": {
    "name": "Factory LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj"
  }
}
```

#### Esempio di request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/quotation \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "simple",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "BRL",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
    "boxes": [{"height": 10, "width": 10, "depth": 10, "weight": 1, "price": 100}]
  }'
```

#### Esempio di response
```json
{
  "status": "success",
  "message": "Cotação Realizada com sucesso!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2021-09-28 14:28:10",
    "object": "package",
    "type": "advanced",
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "freight": "181.95",
        "tax": "28153.894",
        "currency_payment_amount": "24288.1"
      }
    ]
  }
}
```

#### Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `400` | Vettore indisponibile o regola negata | Adeguare paese, peso, scatola o contratto |
| `422` | Validazione | Correggere campi obbligatori |
| `406` | Struttura incompatibile | Rivedere payload e versione |
| `500` | Errore imprevisto | Contattare il team responsabile |

## Osservazioni finali
- La documentazione OpenAPI della v2 e la validazione della request non sono identiche in tutti i dettagli. Validare con il team responsabile dell'API.
- Per consultazione dettagliata della quotazione, la fonte analizzata non porta endpoint v2 equivalente a `GET /v1/quotation/{quotation}/{level}`.
- Tutte le azioni di quotazione e spedizione nella seconda versione possono essere monitorate dalle altre API della prima versione.
