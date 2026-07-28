# Spedizione V2

- [Torna al README della lingua](README.md)

## Panoramica
- Crea spedizioni con contratto semplificato.
- La v2 non espone, nella fonte analizzata, endpoint di status, documenti, annullamento o aggiornamento.
- Versione documentata: v2.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header della lingua: `lang`.

## Convenzioni
- `Content-Type: application/json`
- `quotation` deve essere stata creata nella v2.
- `payment.type` continua a definire il flusso finanziario.

## Flusso di utilizzo
1. Crea la quotazione nella v2.
2. Invia la spedizione con il `quotation` restituito.
3. Monitora il resto del flusso usando gli endpoint della v1, se applicabile al contratto legacy.

## Endpoints
### Creare spedizione
- Metodo e rotta: `POST /v2/shipping`
- Quando usare: nuovo flusso di shipping.
- Regole importanti: la quotazione deve essere v2; `payment.type` deve essere consentito per l'utente.

#### Differenze rispetto alla v1
- La v2 mantiene l'endpoint di creazione, ma la documentazione pubblica non mostra gli altri endpoint del ciclo.
- `attachments` nella v2 accetta solo `PDF` o `XML` secondo la fonte analizzata.
- Il contratto è più rigoroso per l'uso dei campi di indirizzo.

#### Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | sì | ID della quotazione | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Deve essere v2 |
| `carrier_code` | body | integer | sì | Contratto | `1` | Deve esistere |
| `address_sender` | body | object | sì | Mittente | - | Accetta campi manuali o `address_code` |
| `address_receiver` | body | object | sì | Destinatario | - | Accetta campi manuali o `address_code` |
| `payment.type` | body | string | sì | Tipo di pagamento | `billed` | `billed`, `gateway`, `afterward` |
| `attachments` | body | array | no | Allegati | - | PDF/XML nella fonte |
| `biller` | body | object | no | Fatturante | - | Opzionale |
| `nfkey` | body | string | no | Chiave NF-e | - | 44 cifre |

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
    "name": "John Doe",
    "email": "johndoe@example.com",
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
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Warehouse",
    "street": "Main Street",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida",
    "description": "Office",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
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
  },
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ]
}
```

#### Esempio di request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

## Payload minimo e massimo
- Il payload minimo richiede solo i campi indispensabili per identificazione della quotazione, vettore, indirizzi e pagamento.
- Il payload massimo include NF-e, allegati, fatturante e riferimento commerciale.

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `422` | Validazione | Adeguare contratto e campi obbligatori |
| `400` | Regola di business | Verificare eleggibilità della quotazione e del pagamento |
| `500` | Errore imprevisto | Contattare il team responsabile |

## Osservazioni finali
- La v2 documentata nella fonte è snella; per supporto operativo di documenti e tracciamento, la fonte analizzata punta alla v1.
- Tutte le azioni di quotazione e spedizione nella seconda versione possono essere monitorate dalle altre API della prima versione.
