# Manifesto

- [Torna al README della lingua](README.md)

## Panoramica
- Esegue il flusso sincrono di quotazione + spedizione.
- Dopo la creazione, può restituire i documenti finali dell'operazione.
- Associato al flusso operativo principale.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header della lingua: `lang`.

## Convenzioni
- Query opzionale `format`: `pdf` o `zpl`.
- Il flusso usa `billed` come tipo di pagamento nel contratto pubblicato.

## Flusso di utilizzo
1. Invia i dati completi del manifesto.
2. L'API esegue la quotazione, crea la spedizione e genera documenti.
3. Se `format` è `pdf` o `zpl`, la risposta favorisce quel formato.

## Endpoints
### Creare manifesto
- Metodo e rotta: `POST /v1/manifest`
- Query params: `format`.

### Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `format` | query | string | no | Tipo di ritorno | `pdf` | `pdf` o `zpl` |
| `object` | body | string | sì | Oggetto | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | sì | Finalità | `Commercial` | Viene dal flusso di business |
| `tax` | body | string | sì | Imposta | `sender` | `sender` o `receiver` |
| `insurance` | body | boolean | sì | Assicurazione | `true` | - |
| `residential_delivery` | body | boolean | sì | Consegna residenziale | `false` | - |
| `non_stackable` | body | boolean | sì | Non impilabile | `false` | - |
| `measurement` | body | string | sì | Sistema di misura | `metric` | `metric` o `imperial` |
| `ior` | body | boolean | sì | IOR | `false` | - |
| `currency_quote` | body | string | sì | Valuta della quotazione | `USD` | - |
| `currency_payment` | body | string | sì | Valuta di pagamento | `EUR` | - |
| `address_sender` | body | object | sì | Mittente | - | Manuale o `address_code` |
| `address_receiver` | body | object | sì | Destinatario | - | Manuale o `address_code` |
| `boxes` | body | array | sì | Scatole | - | Obbligatorio |
| `items` | body | array | sì | Articoli | - | Obbligatorio eccetto `doc` |
| `payment.type` | body | string | sì | Pagamento | `billed` | Manifesto accetta solo fatturato |
| `declared_freight` | body | object | no | Nolo dichiarato | - | Opzionale |

### Payload minimo
```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Florida",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Payload massimo
```json
{
  "format": "pdf",
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "carrier_code": 1,
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Manifesto de teste",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
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
    "federal_tax_id": "12345678901",
    "state_tax_id": "1234567890",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
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
    "federal_tax_id": "7418529630",
    "state_tax_id": "7418529586",
    "foreign": true
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "Test Product",
      "country_code": "US",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100,
      "selling_unit_value": 110
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Esempio di response
```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "000000",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    }
  }
}
```

## Regole di business
- Il manifesto funziona solo con pagamento fatturato.
- Il documento finale può essere PDF, ZPL o entrambi, secondo il flusso e il vettore.
- Se la quotazione è incompatibile con la versione, l'esecuzione fallisce.

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `400` | Errore del vettore o regola di business | Adeguare payload o verificare ritorno del vettore |
| `422` | Validazione | Correggere campi obbligatori |
| `500` | Errore imprevisto | Contattare il team responsabile |

## Osservazioni finali
- Questo endpoint è sincrono e concentra più responsabilità rispetto ai flussi separati di quotazione + spedizione.
- Raccomandato solo per integrazioni che hanno davvero bisogno del flusso completo in una chiamata.
