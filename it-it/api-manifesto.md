# Manifesto

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Esegue **quotazione + spedizione + generazione dei documenti in un'unica chiamata sincrona**.
- La risposta contiene già il numero di tracciamento e i documenti (AWB, e gli altri quando applicabili).
- È il flusso indicato per le operazioni che hanno bisogno dell'etichetta **immediatamente**, senza attendere
  un'elaborazione asincrona.

> ⚠️ **Il manifesto è sincrono e irreversibile.** Poiché quota e spedisce nella stessa richiesta, non esiste
> una fase intermedia di revisione. Valida bene il payload prima di inviarlo.
>
> ⚠️ **Accetta solo il pagamento fatturato** (`payment.type: billed`).

## Base URL
- Produzione: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Documentazione interattiva: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticazione
```
Authorization: Bearer IL_TUO_TOKEN
Content-Type: application/json
lang: pt
```

## Manifesto vs. quotazione + spedizione

| Tema | Quotazione + Spedizione | Manifesto |
|---|---|---|
| Chiamate | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Scelta del corriere | La scegli tu nell'elenco restituito | La risolve la piattaforma (o la fissi con `carrier_code`/`slug`) |
| Documenti | Consultazione successiva su `/docs` | **Arrivano nella risposta** |
| Pagamento | `billed`, `gateway`, `afterward` | **Solo `billed`** |
| Tipo di risposta | Asincrono | **Sincrono** |
| Revisione prima della spedizione | Sì | No |
| Campi obbligatori | Meno (divisi in 2 fasi) | **Più** (tutto in una volta) |

Usa il manifesto quando: l'operazione è ad alto volume, il corriere è già definito da una regola di
business e devi stampare l'etichetta sul momento.

## Flusso di utilizzo
1. Componi il payload completo (indirizzi, scatole, articoli, pagamento).
2. `POST /v1/manifest?format=pdf` (o `zpl`).
3. Stampa l'etichetta ricevuta in `data.docs`.
4. Usa il `data.shipping` per il [tracciamento](api-rastreio.md), i [documenti](api-documentos.md) e la
   [pianificazione del ritiro](api-agendamento.md).

## Endpoint: creare un manifesto

- **Metodo e rotta:** `POST /v1/manifest`

### Parametri di query

| Nome | Posizione | Tipo | Obbligatorio | Predefinito | Valori | Descrizione |
|---|---|---|---|---|---|---|
| `format` | query | string | no | tutti | `pdf`, `zpl` | Formato dei documenti nella risposta. Vuoto restituisce tutti i formati disponibili |

> `format` può essere inviato anche **nel corpo** del payload. Quando viene inviato in entrambi i punti,
> **prevale il valore del corpo**.

### Parametri — radice del payload

| Campo | Tipo | Obbligatorio | Min. | Max. | Valori / Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `object` | string | **sì** | - | - | `doc`, `package`, `pallet`, `not_doc` | Cosa viene spedito |
| `type` | string | **sì** | - | 30 | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalità della spedizione |
| `tax` | string | **sì** | - | - | `sender`, `receiver` | Chi paga le imposte |
| `insurance` | boolean | **sì** | - | - | - | Attivare l'assicurazione |
| `residential_delivery` | boolean | **sì** | - | - | - | Consegna a un indirizzo residenziale |
| `non_stackable` | boolean | **sì** | - | - | - | Colli non impilabili |
| `measurement` | string | **sì** | - | - | `metric`, `imperial` | Sistema di misura |
| `ior` | boolean | **sì** | - | - | - | Importer of Record |
| `currency_quote` | string | **sì** | 3 | 3 | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta della quotazione |
| `currency_payment` | string | **sì** | 3 | 3 | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta del pagamento |
| `address_sender` | object | **sì** | - | - | - | Mittente (vedi tabella) |
| `address_receiver` | object | **sì** | - | - | - | Destinatario (vedi tabella) |
| `boxes` | array | **sì** | 1 elemento | - | - | Colli (vedi tabella) |
| `payment` | object | **sì** | - | - | - | Pagamento (vedi tabella) |
| `items` | array | condizionale | - | - | - | Obbligatorio quando `object` **non** è `doc` e nessuna scatola usa `kit`. **Vietato quando `object` è `doc`** |
| `carrier_code` | integer | no | - | - | - | Fissa il contratto da usare |
| `slug` | string | no | - | 20 | - | Fissa il contratto tramite l'identificatore testuale |
| `channel_code` | integer | no | - | - | - | Codice del canale di vendita registrato |
| `channel_identifier_number` | string | no | - | - | - | Il tuo identificatore commerciale. **Consigliato** |
| `nfkey` | string | no | 44 | 44 | - | Chiave della NF-e |
| `description` | string | no | 5 | 155 | - | Descrizione libera del manifesto |
| `declared_freight` | object | no | - | - | - | Nolo dichiarato (vedi tabella) |
| `biller` | object | no | - | - | - | Soggetto fatturante mostrato sull'invoice |
| `manufacturer` | object | no | - | - | - | Produttore mostrato sull'invoice |
| `extra` | object | no | - | - | - | Dati ausiliari in coppie chiave/valore (vedi tabella) |
| `format` | string | no | - | - | `pdf`, `zpl` | Alternativa alla query string |
| `size` | string | no | - | - | `label`, `letter` | Dimensione dell'etichetta |

### Parametri — `address_sender` e `address_receiver`

Diversamente dalla spedizione v1/v2, qui l'indirizzo viene indicato **completo**, perché la quotazione avviene
nella stessa chiamata.

**Modo A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questo modo | Codice dell'indirizzo registrato. **Non inviare nessun altro campo di indirizzo insieme** |

**Modo B — indirizzo manuale:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `country_code` | string | **sì** | 2 | 2 | - | Paese in ISO 3166-1 alpha-2 |
| `city` | string | condizionale | 1 | 100 | - | Città. Obbligatoria se `postal_code` non viene indicato |
| `postal_code` | string | condizionale | - | 12 | - | Codice postale/ZIP. Obbligatorio se `state` e `state_code` non vengono indicati |
| `state` | string | condizionale | - | 100 | - | Nome dello stato. Obbligatorio se `state_code` e `postal_code` non vengono indicati |
| `state_code` | string | condizionale | 1 | 10 | - | Sigla dello stato, validata rispetto al paese. Obbligatoria se `state` e `postal_code` non vengono indicati |
| `street` | string | **sì** | 5 | 70 | - | Indirizzo (via) |
| `name` | string | **sì** | 1 | 70 | - | Nome del contatto/ragione sociale |
| `email` | string | **sì** | - | 60 | - | E-mail valida |
| `type` | string | **sì** | - | - | `pj` | `pf` o `pj` |
| `foreign` | boolean | **sì** | - | - | - | Indica se la persona è straniera (usato per validare il `federal_tax_id`) |
| `federal_tax_id` | string | condizionale | - | 20 | - | **Obbligatorio quando il paese è `BR` o `AR`** |
| `phone` | string | no | 1 | 25 | - | Telefono |
| `phone_extension` | string | no | - | 5 | - | Interno |
| `number` | string | no | 1 | 6 | - | Numero civico |
| `district` | string | no | 1 | 30 | - | Quartiere |
| `complement` | string | no | 1 | 20 | - | Complemento |
| `description` | string | no | 1 | 155 | - | Etichetta interna dell'indirizzo |
| `state_tax_id` | string | no | - | 20 | - | Partita IVA statale |
| `company_name` | string | no | 5 | 50 | - | **Solo in `address_sender`.** Ragione sociale richiesta da alcune spedizioni FedEx |

### Parametri — `boxes[]`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `kit` | string | alternativa agli altri modi | - | 50 | SKU del kit registrato. Compila la scatola **e** gli articoli di quella posizione |
| `box_code` | integer | alternativa | - | - | Codice della scatola registrata ([Scatola](api-caixa-v1.md)) |
| `box_identifier` | string | alternativa | - | 50 | Identificatore della scatola registrata |
| `name` | string | **sì** senza `kit`/`box_code`/`box_identifier` | 5 | 155 | Nome della scatola |
| `height` | number | **sì** senza `kit`/`box_code`/`box_identifier` | 1 | - | Altezza (cm con `metric`, in con `imperial`) |
| `width` | number | **sì** senza `kit`/`box_code`/`box_identifier` | 1 | - | Larghezza |
| `depth` | number | **sì** senza `kit`/`box_code`/`box_identifier` | 1 | - | Profondità |
| `weight` | number | **sì** quando `object` è `doc` | 0.001 | - | Peso del collo (kg con `metric`, lb con `imperial`) |
| `price` | number | **sì** quando `object` è `doc` | 0 | - | Valore dichiarato del collo |

### Parametri — `items[]`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `box_id` | integer | **sì** | - | - | **Posizione** della scatola nell'array `boxes` (1 = prima) |
| `quantity` | integer | **sì** | 1 | 99 | Quantità dell'articolo |
| `sku` | string | alternativa ai campi manuali | 1 | 50 | SKU dell'articolo registrato ([Articoli](api-itens-v1.md)) |
| `name` | string | **sì** senza `sku` | 6 | 50 | Nome dell'articolo |
| `country_code` | string | **sì** senza `sku` | 2 | 2 | Paese di origine dell'articolo |
| `hscode` | string | **sì** senza `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **sì** senza `sku` | 0.001 | 99 | Peso unitario |
| `unit_value` | number | **sì** senza `sku` | 0 | - | Valore unitario |
| `description` | string | no | 5 | 450 | Descrizione dettagliata dell'articolo |
| `selling_unit_value` | number | condizionale | 0 | - | Valore di vendita unitario. **Necessario quando `ior` è `true`** |

> **Attenzione alle differenze rispetto alla quotazione:** qui `quantity` ha un massimo di **99** (non 1000) e
> `description` accetta fino a **450** caratteri (non 155).

### Parametri — `payment`

| Campo | Tipo | Obbligatorio | Valori | Descrizione |
|---|---|---|---|---|
| `payment` | object | **sì** | - | Involucro |
| `payment.type` | string | **sì** | **`billed`** | Il manifesto accetta **solo** il pagamento fatturato |

### Parametri — `declared_freight`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `declared` | boolean | no | - | - | Abilita il nolo dichiarato. Con `true`, il valore può essere `0` e le imposte vengono calcolate con nolo dichiarato pari a zero |
| `value` | number | no | 0 | - | Valore del nolo dichiarato |
| `currency` | string | **sì** quando `value` è indicato | 3 | 3 | Valuta del nolo dichiarato |

### Parametri — `extra`

Coppie chiave/valore ausiliarie, trasmesse al processo del manifesto.

| Regola | Dettaglio |
|---|---|
| Struttura | Oggetto **piatto** (un livello, senza annidamento) |
| Chiavi | Solo lettere minuscole e `_` |
| Valori | Testo, intero o decimale |

| Chiave conosciuta | Tipo | Regola | Descrizione |
|---|---|---|---|
| `federal_tax_provision` | number | - | Accantonamento dell'imposta federale |
| `state_tax_provision` | number | - | Accantonamento dell'imposta statale |
| `usd_rate_provision` | number | > 0 | Accantonamento del cambio del dollaro |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Parametri — `biller` e `manufacturer`

- `biller`: stesso formato descritto in [Spedizione (v1)](api-remessa-v1.md#parametri--biller-soggetto-fatturante).
  Nel manifesto, `biller.name` accetta fino a **355** caratteri.
- `manufacturer`: stesso formato descritto in [Quotazione (v1)](api-cotacao-v1.md#parametri--manufacturer).

### Payload minimo

Manifesto di un documento (`object: doc`) — in questo caso `items` è vietato:

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
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "name": "Negozio Esempio SRL",
    "email": "contato@example.com",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Busta documenti",
      "height": 2,
      "width": 23,
      "depth": 32,
      "weight": 0.3,
      "price": 50
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
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "slug": "fedex-br",
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Manifesto del lotto di marzo",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Centro di distribuzione SP",
    "street": "Via Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Capannone 2",
    "company_name": "Negozio Esempio Commercio SRL",
    "name": "Negozio Esempio SRL",
    "email": "contato@example.com",
    "phone": "11989677465",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami",
    "description": "Ufficio Miami",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Scatola master 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "T-shirt cotone S",
      "description": "T-shirt 100% cotone, manica corta, colore bianco, taglia S",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Scarpe sportive",
      "description": "Scarpe da corsa uomo numero 44",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9
    }
  ],
  "biller": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "number": "100",
    "name": "Holding Esempio SRL",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabbrica Esempio SRL",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "type": "pj"
  },
  "payment": {
    "type": "billed"
  },
  "extra": {
    "federal_tax_provision": 10,
    "state_tax_provision": 7,
    "usd_rate_provision": 5.42
  }
}
```

### Esempio di request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
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
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Via Principale", "name": "Negozio Esempio SRL", "email": "contato@example.com", "type": "pj", "federal_tax_id": "12345678000199", "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "type": "pf", "foreign": true},
    "boxes": [{"name": "Busta documenti", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Test",
    "tracking_id": "271249639538",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.shipping` | string (UUID) | **Identificatore della spedizione.** Usalo per tracciamento, documenti e pianificazione |
| `data.carrier` | string | Corriere che ha elaborato la spedizione |
| `data.tracking_id` | string | Numero di tracciamento (AWB) |
| `data.docs` | object | Documenti generati, nel formato `nome → { url, base64 }` |

> Documenti aggiuntivi (invoice, packing list, dichiarazione) possono essere generati in background.
> Consulta [Documenti della Spedizione](api-documentos.md) qualche istante dopo per ottenere l'elenco completo.

## Errore del corriere

Quando il corriere rifiuta la spedizione, la risposta è `400` e contiene il suo riscontro grezzo:

```json
{
  "status": "error",
  "message": "[1234] CARRIER - Carrier error message",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Campo | Descrizione |
|---|---|
| `message` | `[codice] CORRIERE - messaggio` restituito dal corriere |
| `data` | Payload grezzo del corriere. **La struttura varia da corriere a corriere** |

> Registra `message` e `data` nei tuoi log: è l'unica fonte del motivo tecnico del rifiuto.

## Regole di business
- **Pagamento:** solo `billed`. Qualsiasi altro valore viene rifiutato in validazione.
- **`items` e `object: doc`:** con `object: doc`, il campo `items` **non può** essere inviato.
  Con qualsiasi altro `object`, `items` è obbligatorio — tranne quando una scatola usa `kit`.
- **Kit:** quando una scatola indica `kit`, la scatola e gli articoli di quella posizione provengono dalla registrazione,
  e `boxes[].weight`/`boxes[].price` vengono ignorati.
- **`state_code`:** validato rispetto al `country_code` dello stesso indirizzo.
- **`federal_tax_id`:** obbligatorio per `BR` e `AR`.
- **`address_code`:** quando lo usi, **nessun** altro campo di indirizzo può essere inviato nello stesso oggetto.
- **Le preimpostazioni dell'account** vengono applicate come nella bozza: il payload vince sempre sul preset.
  Indicare `declared_freight` scarta il preset del nolo dichiarato; indicare
  `address_sender.country_code` scarta il preset dell'indirizzo del mittente.
- **`not_doc`** è accettato per compatibilità e convertito internamente in `package`.

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Errore del corriere o regola di business | Leggere `message` e `data`; adeguare il payload o contattare il supporto |
| `403` | Permesso mancante per creare un manifesto | Richiedere il permesso al supporto |
| `422` | Validazione dei campi | Correggere i campi elencati in `data` |
| `500` | Errore inatteso | **Prima di ripetere**, verifica in [Tracciamento](api-rastreio.md) se la spedizione è stata creata |

> ⚠️ **Non ripetere mai un manifesto automaticamente dopo un `500`.** Poiché la chiamata crea la spedizione per davvero,
> una ripetizione può generare una spedizione duplicata. Verifica prima tramite il `channel_identifier_number`.

## Note finali
- Indica sempre `channel_identifier_number`: è la tua protezione contro i duplicati e il modo per localizzare
  la spedizione se la risposta va perduta.
- Il manifesto concentra molta responsabilità in una sola chiamata. Per le integrazioni che possono attendere,
  il flusso di [quotazione](api-cotacao-v1.md) + [spedizione](api-remessa-v1.md) è più sicuro.
- Per il flusso opposto (payload parziale, validato in seguito), vedi [Bozza](api-rascunho.md).
