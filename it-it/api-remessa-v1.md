# Spedizione

- [Torna al README della lingua](README.md)

## Panoramica
- Crea, aggiorna, annulla e consulta spedizioni.
- Espone anche consultazione di documenti e status/tracciamento.
- È la versione legacy più completa del flusso di shipping.
## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header opzionale della lingua: `lang`.

## Convenzioni
- `Content-Type: application/json`
- La spedizione usa `quotation` come collegamento con la quotazione.
- Il `quotation` deve essere della stessa versione dell'API di spedizione.
- `payment.type` definisce il modello finanziario dell'invio.

## Flusso di utilizzo
1. Genera la quotazione.
2. Verifica se è eleggibile per spedizione.
3. Invia la spedizione con il `quotation` restituito.
4. Consulta `status` fino a quando la spedizione viene processata.
5. Scarica i documenti in `/docs`.
6. Se necessario, programma il ritiro.

## Regole di business
- Quotazione `package`/`pallet` con tipo `simple` non è eleggibile per spedizione.
- La quotazione non può essere scaduta.
- La spedizione non può essere stata creata precedentemente per la stessa quotazione.
- Il tipo di pagamento deve essere abilitato per l'utente.
- Per DHL con mittente PJ in Brasile, `nfkey` può essere obbligatoria.
- Dopo la creazione, la spedizione è irreversibile nell'operazione normale.
- Modifiche successive sono ammesse solo in casi eccezionali: invio del numero della nota fiscale quando non è stato informato e invio di documenti da allegare alla spedizione quando non sono stati inviati durante la creazione.

## Endpoints
### Creare spedizione
- Metodo e rotta: `POST /v1/shipping`
- Quando usare: dopo avere ottenuto una quotazione eleggibile.

#### Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | sì | ID della quotazione | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Deve essere della prima versione |
| `carrier_code` | body | integer | sì | Contratto/vettore | `1` | Deve esistere per la quotazione |
| `address_sender` | body | object | sì | Mittente | - | Può usare `address_code` o campi manuali |
| `address_receiver` | body | object | sì | Destinatario | - | Può usare `address_code` o campi manuali |
| `payment.type` | body | string | sì | Tipo di pagamento | `billed` | `billed`, `gateway`, `afterward` |
| `type` | body | string | no | Finalità dell'invio | `Commercial` | Opzionale |
| `nfkey` | body | string | no | Chiave NF-e | `12345678901234567890123456789012345678901234` | 44 cifre |
| `invoice_number` | body | string | no | Numero della fattura | `NF-123` | Se non inviato, usa il codice interno |
| `shipment_reference` | body | string | no | Riferimento dell'invio | `REF-2024-01` | Se non inviato, usa l'ID interno |
| `attachments` | body | array | no | Allegati | - | PDF, XML, JPEG, PNG |
| `biller` | body | object | no | Fatturante | - | Usato nella invoice |

#### Payload minimo
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj"
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "Doe Jhon",
    "email": "doejhon@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

#### Payload massimo
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "nfkey": "12345678901234567890123456789012345678901234",
  "type": "Commercial",
  "invoice_number": "NF-123",
  "shipment_reference": "REF-2024-01",
  "carrier_code": 1,
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ],
  "address_sender": {
    "address_code": 1
  },
  "address_receiver": {
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Factory LTDA",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj",
    "foreign": false
  },
  "payment": {
    "type": "gateway"
  }
}
```

#### Esempio di request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "Doe Jhon", "email": "doejhon@example.com", "phone": "11992494597", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

#### Esempio di response
```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

### Consultare status della spedizione
- Metodo e rotta: `GET /v1/shipping/{shipping}/status`
- `shipping` accetta: UUID della quotazione, AWB, o chiave NF-e di 44 cifre.

### Consultare status per canale
- Metodo e rotta: `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Consultare documenti
- Metodo e rotta: `GET /v1/shipping/{shipping}/docs`
- Restituisce documenti, allegati e, se consentito, docs incorporati.

### Consultare documenti per canale
- Metodo e rotta: `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Aggiornare spedizione
- Metodo e rotta: `PUT /v1/shipping/{shipping}`
- Usa per allegare documento e/o collegare NF-e in caso eccezionale.
- I file allegati devono essere in `base64` e accettare solo `PDF`, `XML`, `JPEG` o `PNG`.
- La dimensione massima è di `5 MB` per file.

#### Esempio di request
```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "attachments": [
      {
        "description": "Nota Fiscal",
        "base64": "JVBERi0xLjcK..."
      }
    ]
  }'
```

#### Esempio di response
```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Annullare spedizione
- Metodo e rotta: `DELETE /v1/shipping/{shipping}`
- `reason` è opzionale nel body e accetta testo libero.

#### Esempio di request
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Customer requested cancellation"}'
```

#### Esempio di response
```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

### Rimuovere allegati
- Metodo e rotta: `DELETE /v1/shipping/{shipping}/docs`
- Invia `attachments` con `id` o `name`.

#### Esempio di request
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

#### Esempio di response
```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!"
}
```

## Consultazioni e documenti
### Parametri di documenti
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `attachments` | query | boolean | no | Include allegati | `true` | Default `true` |
| `embedded` | query | boolean | no | Include docs incorporati | `true` | Default `true` |
| `base64` | query | boolean | no | Include base64 | `true` | Default `true` |
| `documents` | query | string | no | Filtra tipi | `invoice_pdf,awb_pdf` | Lista separata da virgola |

### Tipi di documenti identificati
- `awb_pdf`
- `awb_zpl`
- `awb_prc_pdf`
- `awb_prc_zpl`
- `awb_prc_fedex_pdf`
- `awb_prc_fedex_zpl`
- `declaration_pdf`
- `invoice_pdf`
- `non_dangerous_pdf`
- `packing_list_pdf`

### Esempio di consultazione documenti
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `400` | Spedizione non trovata o regola di business | Verificare `quotation`, `carrier_code` ed eleggibilità |
| `401` | Pagamento non autorizzato | Adeguare la configurazione dell'account |
| `404` | Risorsa non trovata | Confermare l'identificatore informato |
| `422` | Validazione | Correggere payload |
| `500` | Errore imprevisto | Contattare il team responsabile |

## Osservazioni finali
- La spedizione v1 è il punto di integrazione più completo del ciclo operativo.
- Consulta `api-documentos.md` e `api-rastreio.md` per esempi dedicati di consultazione e monitoraggio.
