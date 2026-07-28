# Bozza

- [Torna al README della lingua](README.md)

## Panoramica
- Crea, elenca, aggiorna e rimuove bozze di spedizione.
- La bozza viene processata in modo asincrono e può evolvere in quotazione + spedizione.
- Associato al flusso operativo principale.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header della lingua: `lang`.

## Convenzioni
- La bozza accetta payload parziali.
- Lo `status` della bozza può essere `received`, `edited`, `validating`, `incomplete`, `complete`, `processing` o `error`.

## Flusso di utilizzo
1. Invia la bozza con i dati che già possiedi.
2. Attendi il processamento asincrono.
3. Elenca o consulta le bozze.
4. Aggiorna solo bozze in status `incomplete`.
5. Rimuovi bozze quando necessario.

## Endpoints
### Creare bozza
- Metodo e rotta: `POST /v1/draft`

### Aggiornare bozza
- Metodo e rotta: `PUT /v1/draft/{code}`
- Accetta solo modifica di bozza `incomplete`.

### Elencare bozze
- Metodo e rotta: `GET /v1/draft`
- Può filtrare per `drafts.drafts_ids`.

### Rimuovere bozza
- Metodo e rotta: `DELETE /v1/draft`

### Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | no | Canale | `1` | - |
| `channel_identifier_number` | body | string | no | Identificatore commerciale | `334455` | - |
| `carrier_code` | body | integer | no | Vettore preferito | `1` | - |
| `object` | body | string | no | Oggetto | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | no | Tipo | `Commercial` | - |
| `tax` | body | string | no | Imposta | `sender` | - |
| `insurance` | body | boolean | no | Assicurazione | `true` | - |
| `measurement` | body | string | no | Misure | `metric` | Default può venire da preset |
| `currency_quote` | body | string | no | Valuta della quotazione | `USD` | - |
| `currency_payment` | body | string | no | Valuta del pagamento | `BRL` | - |
| `address_sender` | body | object | no | Mittente | - | Accetta compilazione parziale |
| `address_receiver` | body | object | no | Destinatario | - | Accetta compilazione parziale |
| `boxes` | body | array | no | Scatole | - | Opzionale |
| `items` | body | array | no | Articoli | - | Opzionale |
| `payment.type` | body | string | no | Tipo di pagamento | `billed` | `billed`, `afterward` |

### Esempio di request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP", "city": "Diadema"},
    "address_receiver": {"country_code": "US", "city": "Florida"},
    "boxes": [{"height": 10, "width": 10, "depth": 10}],
    "payment": {"type": "billed"}
  }'
```

### Esempio di response
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno"
}
```

## Regole di business
- Una bozza senza payload utile restituisce warning.
- Il backend usa preset dell'utente quando disponibili.
- Le bozze possono essere convertite in quotazione e spedizione tramite processamento asincrono.

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `400` | Bozza vuota, non trovata o non modificabile | Inviare dati validi o scegliere un'altra bozza |
| `422` | Validazione | Correggere payload |
| `500` | Errore imprevisto | Contattare il team responsabile |

## Osservazioni finali
- La bozza è l'opzione migliore quando l'integrazione non ha ancora tutti i dati finali.
- Consulta `api-cotacao-v1.md` e `api-remessa-v1.md` per capire cosa genererà il processamento asincrono.
