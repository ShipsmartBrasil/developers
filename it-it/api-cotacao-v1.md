# Quotazione

- [Torna al README della lingua](README.md)

## Panoramica
- Genera quotazioni internazionali con dettaglio di nolo, assicurazione e imposte.
- Usa quando devi simulare l'invio prima di creare la spedizione.
- La quotazione è la base per spedizione, tracciamento e documenti.
## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Tipo: Bearer token.
- Header: `Authorization: Bearer SEU_TOKEN`
- Header aggiuntivo accettato: `lang: pt|en|es`.
- La lingua predefinita è `pt`.

## Convenzioni
- `Content-Type: application/json`
- Date nel formato `Y-m-d H:i:s` nelle risposte.
- Valute in codice ISO 4217 a 3 lettere.
- `level` controlla il dettaglio del risultato: `simple` o `detailed`.
- Status HTTP comuni: `200`, `400`, `404`, `406`, `422`, `500`.

## Flusso di utilizzo
1. Invia una quotazione con origine, destinazione, oggetto, tipo e articoli/scatole.
2. Ricevi il `quotation` restituito dall'API.
3. Consulta il risultato con `GET /v1/quotation/{quotation}/{level}`.
4. Se la quotazione è eleggibile, usa lo stesso `quotation` per creare la spedizione.

## Differenze di contratto
| Endpoint | Differenza principale | Impatto |
|---|---|---|
| `POST /v1/quotation` | Crea la quotazione | Flusso iniziale |
| `GET /v1/quotation/{quotation}/{level}` | Consulta risultato | Permette di rivedere carriers |

## Argomenti funzionali
### Quotazione semplice, avanzata e `items`
- `simple`: calcola per scatola con peso/valore diretto. Per `doc`, è il flusso naturale.
- `advanced`: calcola con scatole e articoli distribuiti per `box_id`.
- `items`: la validazione del backend accetta questo tipo, ma il contratto OpenAPI non lo espone chiaramente. Validare con il team responsabile dell'API.

### `package`, `pallet` e `doc`
- `doc`: documento. La quotazione semplice per documenti usa scatola con peso e prezzo.
- `package`: pacco/scatola.
- `pallet`: pallet. Può usare `volume` invece di `height/width/depth`.

### Quotazione eleggibile per spedizione
- La quotazione deve esistere, non essere scaduta, appartenere alla stessa `api_version` della spedizione e non essere ancora stata usata.
- `package`/`pallet` con tipo `simple` non sono eleggibili per spedizione.
- La spedizione richiede anche contratto compatibile e tipo di pagamento consentito.

## Endpoints
### Creare quotazione
- Obiettivo: calcolare nolo, assicurazione e imposte.
- Metodo e rotta: `POST /v1/quotation`
- Quando usare: prima di creare spedizione.
- Regole importanti: origine e destinazione obbligatorie; `currency_quote` e `currency_payment` obbligatorie; `measurement` accetta `metric` o `imperial`.

#### Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | no | Canale di vendita | `1` | Deve esistere nel cadastro |
| `channel_identifier_number` | body | string | no | Identificatore del canale | `334455` | Libero |
| `object` | body | string | sì | Oggetto inviato | `package` | `doc`, `package`, `pallet`, `not_doc` nella fonte |
| `type` | body | string | sì | Tipo della quotazione | `advanced` | `simple`, `advanced`, `items` nella validazione |
| `tax` | body | string | sì | Chi paga le imposte | `sender` | `sender` o `receiver` |
| `insurance` | body | boolean | sì | Assicurazione | `true` | - |
| `measurement` | body | string | no | Sistema di misura | `metric` | `metric` o `imperial` |
| `residential_delivery` | body | boolean | no | Consegna residenziale | `false` | - |
| `non_stackable` | body | boolean | no | Indica non impilabile | `false` | - |
| `ior` | body | boolean | no | Importer of Record | `false` | - |
| `currency_quote` | body | string | sì | Valuta della quotazione | `BRL` | ISO 4217 |
| `currency_payment` | body | string | sì | Valuta di pagamento | `BRL` | ISO 4217 |
| `address_sender` | body | object | sì | Origine | - | Può usare `address_code` o campi manuali |
| `address_receiver` | body | object | sì | Destinazione | - | Può usare `address_code` o campi manuali |
| `boxes` | body | array | sì | Scatole/pacchi | - | Obbligatorio salvo quando `type=items` |
| `items` | body | array | no | Articoli del carico | - | Obbligatorio in `advanced`/`items` |
| `manufacturer` | body | object | no | Produttore nella invoice | - | Opzionale |
| `declared_freight` | body | object | no | Nolo dichiarato | `{"declared":true,"value":0,"currency":"USD"}` | Solo quando applicabile |
| `level` | query | string | no | Livello di dettaglio | `simple` | `simple` o `detailed` |

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
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida"
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
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": false,
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida"
  },
  "boxes": [
    {
      "name": "Caixa 1",
      "height": 20,
      "width": 20,
      "depth": 20
    }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "BR",
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
  --url https://api.shipsmart.global/v1/quotation \
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
    "address_sender": {
      "country_code": "BR",
      "state_code": "SP",
      "postal_code": "09941070",
      "city": "Diadema"
    },
    "address_receiver": {
      "country_code": "US",
      "state_code": "FL",
      "postal_code": "33101",
      "city": "Florida"
    },
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
    "description": null,
    "channel": "Domenic Dooley",
    "channel_identifier_number": "334455",
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "valid_until": "2021-09-28 20:28:10",
        "transit_days": 5,
        "freight": "181.95",
        "insurance": "11.45",
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
| `400` | Fallimento di regola di business | Adeguare peso, CEP, eleggibilità o tipo di quotazione |
| `406` | Payload inconsistente | Verificare campi obbligatori e tipi |
| `422` | Validazione | Correggere campi obbligatori/formati |
| `500` | Errore imprevisto | Ripetere e contattare il team responsabile |

## Consultare quotazione
### Metodo e rotta
- `GET /v1/quotation/{quotation}/{level}`
- `level` obbligatorio nel path: `simple` o `detailed`.

### Esempio
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Response riepilogativa
```json
{
  "status": "success",
  "message": "Resultado da cotação solicitada!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "object": "package",
    "type": "advanced",
    "carriers": []
  }
}
```

## Osservazioni finali
- La risposta può portare `warnings` quando ci sono avvisi di cadastro o di trasporto.
- La fonte analizzata mostra divergenza tra documentazione OpenAPI e validazione in alcuni campi. Validare con il team responsabile dell'API prima di integrare in produzione.
