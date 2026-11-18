# Quotazione (v1)

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Calcola nolo, assicurazione e imposte per una spedizione internazionale, su tutti i corrieri abilitati sul tuo account.
- È il **primo passo obbligatorio** del flusso: la spedizione, il tracciamento, i documenti e la pianificazione del ritiro
  sono tutti legati al codice di quotazione (`quotation`).
- La v1 è la versione completa: accetta un indirizzo dettagliato in origine/destinazione e dispone di un endpoint per riconsultare il risultato.

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
Dettagli in [Primi Passi](api-primeiros-passos.md).

## Flusso di utilizzo
1. `POST /v1/quotation` con origine, destinazione, oggetto, tipo e scatole/articoli.
2. La risposta contiene già il risultato calcolato (`data.carriers`) e il codice `data.quotation`.
3. Se hai bisogno di riconsultare (o di cambiare il livello di dettaglio), usa `GET /v1/quotation/{quotation}/{level}`.
4. Scegli un corriere dall'elenco e usa `code` (o `slug`) + `quotation` per creare la
   [spedizione](api-remessa-v1.md) **prima** del `valid_until`.

```
POST /v1/quotation  ──►  quotation + carriers[]
                              │
                              ▼
                   POST /v1/shipping  (carrier_code | slug)
```

## Concetti essenziali

### `object` — cosa viene spedito
| Valore | Significato | Note |
|---|---|---|
| `doc` | Documento | Quotazione per scatola, indicando `weight` e `price` |
| `package` | Pacco/scatola | Flusso più comune |
| `pallet` | Pallet | Permette di indicare `volume` invece di altezza/larghezza/profondità |
| `not_doc` | Legacy | Mantenuto per compatibilità; viene convertito internamente in `package`. **Non usarlo nelle nuove integrazioni** |

### `type` — come viene costruito il calcolo
| Valore | Cosa invii | Quando usarlo |
|---|---|---|
| `simple` | Solo `boxes[]` con `weight` e `price` | Simulazione rapida del nolo. **Non è eleggibile per la spedizione quando `object` è `package` o `pallet`** |
| `advanced` | `boxes[]` (dimensioni) + `items[]` con `box_id` che indica la posizione della scatola | Flusso standard per la spedizione: controlli la distribuzione degli articoli nelle scatole |
| `items` | Solo `items[]` con `sku` (le scatole sono scelte dalla piattaforma) | Quando articoli e scatole sono già registrati e vuoi che la piattaforma faccia l'incastro |

> **Attenzione:** `type: simple` con `object: package` o `object: pallet` serve solo per la simulazione.
> Al tentativo di creare la spedizione, l'API risponde `Cotação Simples é inelegível para remessa!`.
> Per la spedizione usa `advanced` o `items`.

### `tax` — chi paga le imposte
| Valore | Significato |
|---|---|
| `sender` | DDP: il mittente paga imposte e tasse a destinazione |
| `receiver` | DDU: il destinatario paga imposte e tasse a destinazione |

### `measurement` — sistema di misura
| Valore | Dimensioni | Peso | Volume |
|---|---|---|---|
| `metric` (predefinito) | centimetri | chilogrammi | m³ |
| `imperial` | pollici | libbre | ft³ |

### `level` — livello di dettaglio del risultato
| Valore | Risultato |
|---|---|
| `simple` (predefinito) | Elenco essenziale di corrieri con i totali |
| `detailed` | Include la composizione dettagliata (nolo, assicurazione, imposta, prima e dopo le spese amministrative) |

Inviato come **query string** nel `POST` e come **segmento di path** nel `GET`.

## Endpoint: creare una quotazione

- **Metodo e rotta:** `POST /v1/quotation`
- **Query string opzionale:** `level=simple|detailed`
- **Quando usarlo:** ogni volta che hai bisogno di prezzo, tempi e imposte, e prima di creare qualsiasi spedizione.

### Parametri — radice del payload

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Valori accettati | Descrizione |
|---|---|---|---|---|---|---|---|
| `object` | string | **sì** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | Cosa viene spedito |
| `type` | string | **sì** | - | - | - | `simple`, `advanced`, `items` | Come viene costruito il calcolo |
| `tax` | string | **sì** | - | - | - | `sender`, `receiver` | Chi paga le imposte |
| `insurance` | boolean | **sì** | - | - | - | `true`, `false` | Attivare l'assicurazione |
| `currency_quote` | string | **sì** | 3 | 3 | - | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta in cui viene presentata la quotazione |
| `currency_payment` | string | **sì** | 3 | 3 | - | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta in cui verrà effettuato il pagamento |
| `address_sender` | object | **sì** | - | - | - | - | Origine (vedi tabella sotto) |
| `address_receiver` | object | **sì** | - | - | - | - | Destinazione (vedi tabella sotto) |
| `boxes` | array | condizionale | - | - | - | - | Obbligatorio quando `type` **non** è `items` |
| `items` | array | condizionale | - | - | - | - | Obbligatorio quando `type` **non** è `simple` e nessuna scatola usa `kit` |
| `measurement` | string | no | - | - | `metric` | `metric`, `imperial` | Sistema di misura |
| `ior` | boolean | no | - | - | `false` | `true`, `false` | Importer of Record. Quando è `true`, `items[].selling_unit_value` diventa necessario |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Consegna a un indirizzo residenziale |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Colli non impilabili |
| `description` | string | no | - | 155 | `null` | - | Descrizione libera della quotazione, come tuo riferimento |
| `channel_code` | integer | no | - | - | preset dell'account | - | Codice del canale di vendita registrato sul tuo account |
| `channel_identifier_number` | string | no | - | - | `null` | - | Il tuo identificatore commerciale (n° dell'ordine). **Consigliato**: permette di consultare status/documenti per canale |
| `channel_order_number` | string | no | - | 100 | `null` | - | Numero dell'ordine nel canale di vendita |
| `declared_freight` | object | no | - | - | preset dell'account | - | Nolo dichiarato (vedi tabella sotto) |
| `manufacturer` | object | no | - | - | `null` | - | Produttore mostrato sulla Commercial Invoice (vedi tabella sotto) |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Livello di dettaglio del risultato |

### Parametri — `address_sender` e `address_receiver`

Esistono **due modi** per indicare l'indirizzo. Scegline uno e non mischiarli:

**Modo A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questo modo | Codice di un indirizzo registrato sul tuo account. Quando lo usi, non inviare gli altri campi |

**Modo B — indirizzo manuale:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `country_code` | string | **sì** | 2 | 2 | Paese in ISO 3166-1 alpha-2 (es.: `BR`, `US`) |
| `city` | string | **sì** | - | 100 | Città |
| `postal_code` | string | condizionale | - | 12 | Codice postale/ZIP. Obbligatorio se `state` non viene indicato. **Per gli indirizzi in Brasile è sempre obbligatorio** |
| `state` | string | condizionale | - | - | Nome dello stato/provincia. Obbligatorio se `state_code` e `postal_code` non vengono indicati |
| `state_code` | string | condizionale | - | 10 | Sigla dello stato/provincia (es.: `SP`, `FL`). Obbligatorio se `state` e `postal_code` non vengono indicati |

> **Regola pratica:** indica sempre `country_code` + `city` + (`postal_code` **o** `state_code`).
> Questa combinazione copre tutti i paesi.

### Parametri — `boxes[]`

Ogni elemento dell'array rappresenta un collo. Esistono **tre modi** per descrivere la scatola:

**Modo A — kit registrato** (compila la scatola **e** gli articoli automaticamente):

| Campo | Tipo | Obbligatorio | Max. | Descrizione |
|---|---|---|---|---|
| `kit` | string | **sì** in questo modo | 50 | SKU del kit registrato. Gli altri campi della scatola e gli articoli di quella posizione vengono compilati dalla registrazione del kit |

**Modo B — scatola registrata:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `box_code` | integer | **sì** in questo modo | Codice della scatola registrata ([Scatola](api-caixa-v1.md)) |
| `box_identifier` | string | alternativa a `box_code` | Identificatore della scatola registrata (es.: `CX0236`) |

**Modo C — scatola indicata manualmente:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `name` | string | no | 4 | 60 | Nome/etichetta della scatola |
| `height` | number | **sì** | 1 | - | Altezza |
| `width` | number | **sì** | 1 | - | Larghezza |
| `depth` | number | **sì** | 1 | - | Profondità |
| `volume` | number | condizionale | 0.001 | - | **Solo quando `object` è `pallet`.** Alternativa ad altezza/larghezza/profondità |

**Campi validi in qualunque modo (tranne `kit`):**

| Campo | Tipo | Obbligatorio | Min. | Descrizione |
|---|---|---|---|---|
| `weight` | number | **sì** quando `object` è `doc`/`package` **e** `type` è `simple` | > 0 | Peso del collo |
| `price` | number | **sì** quando `object` è `doc`/`package` **e** `type` è `simple` | > 0 | Valore dichiarato del collo |

> `weight` e `price` vengono **ignorati** quando la scatola usa `kit`.

### Parametri — `items[]`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `quantity` | integer | **sì** | 1 | 1000 | Quantità dell'articolo |
| `unit_value` | number | **sì** | - | - | Valore unitario nella valuta della quotazione |
| `box_id` | integer | **sì**, tranne quando `type` è `items` | - | - | **Posizione** della scatola nell'array `boxes` (1 = prima scatola, 2 = seconda...). Non è il `box_code` |
| `sku` | string | **sì** quando `type` è `items` | - | 50 | SKU dell'articolo registrato ([Articoli](api-itens-v1.md)). Quando indicato, vengono usati i dati della registrazione e gli altri campi possono essere omessi |
| `ean` | string | no | - | 50 | Codice EAN dell'articolo |
| `name` | string | **sì** quando `sku` non è indicato | 6 | 50 | Nome dell'articolo (compare sull'invoice) |
| `description` | string | no | - | 155 | Descrizione dettagliata dell'articolo |
| `country_code` | string | **sì** quando `sku` non è indicato | 2 | 2 | Paese di origine/fabbricazione dell'articolo |
| `hscode` | string | **sì** quando `sku` non è indicato | 6 | 10 | NCM/HS Code dell'articolo. Definisce la tabella delle imposte applicata |
| `weight` | number | **sì** quando `sku` non è indicato | 0.001 | - | Peso unitario dell'articolo |
| `selling_unit_value` | number | condizionale | - | - | Valore di vendita unitario. **Necessario quando `ior` è `true`** |
| `is_groupable` | boolean | no | - | - | Predefinito `true`. Quando è `false`, l'articolo riceve una scatola esclusiva nel raggruppamento `smartbox` |

> **Descrizioni identiche in articoli diversi** generano un `warning` nella risposta
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> La quotazione viene eseguita, ma conviene rivedere le descrizioni per evitare il blocco in dogana.

### Parametri — `declared_freight`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valore del nolo dichiarato |
| `currency` | string | **sì** quando `value` è indicato | 3 | 3 | Valuta del nolo dichiarato. Stesso elenco di `currency_quote` |

### Parametri — `manufacturer`

Dati del produttore stampati sulla Commercial Invoice. Invia `address_code` **oppure** i campi manuali.

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `address_code` | integer | alternativa ai campi manuali | - | - | Codice di un indirizzo registrato. Quando lo usi, gli altri campi sono opzionali |
| `name` | string | **sì** senza `address_code` | 5 | 30 | Ragione sociale/nome del produttore |
| `country_code` | string | **sì** senza `address_code` | 2 | 2 | Paese |
| `state_code` | string | **sì** senza `address_code` | 1 | 10 | Sigla dello stato |
| `state` | string | no | - | 155 | Nome dello stato |
| `postal_code` | string | **sì** senza `address_code` | - | 12 | Codice postale/ZIP |
| `city` | string | **sì** senza `address_code` | - | 100 | Città |
| `street` | string | **sì** senza `address_code` | - | 70 | Indirizzo (via) |
| `number` | string | no | - | 6 | Numero civico |
| `district` | string | no | - | 30 | Quartiere |
| `complement` | string | no | - | 20 | Complemento |
| `email` | string | **sì** senza `address_code` | - | 60 | E-mail |
| `phone` | string | **sì** senza `address_code` | - | 25 | Telefono |
| `phone_extension` | string | no | - | 6 | Interno |
| `type` | string | **sì** senza `address_code` | - | - | `pf` o `pj` |
| `federal_tax_id` | string | no | - | 20 | CPF/CNPJ o equivalente |
| `state_tax_id` | string | no | - | 20 | Partita IVA statale |

> Per gli articoli con `hscode` che inizia per `64` (calzature), il nome del produttore è richiesto dalla dogana.

### Payload minimo

Quotazione di un documento, tipo simple — il payload più piccolo accettato:

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
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 0.5,
      "price": 100
    }
  ]
}
```

### Payload minimo eleggibile per la spedizione

Per generare la spedizione di un pacco è necessario usare `advanced` (o `items`) e dichiarare gli articoli:

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
  },
  "boxes": [
    { "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "T-shirt cotone",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9
    }
  ]
}
```

### Payload massimo

Tutti i campi accettati dall'endpoint:

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "channel_order_number": "334455",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": false,
  "ior": true,
  "description": "Ordine del marketplace, lotto di ottobre",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
  "boxes": [
    {
      "name": "Scatola master",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 2.5,
      "price": 150
    },
    {
      "box_identifier": "CX0236"
    },
    {
      "box_code": 3
    },
    {
      "kit": "KIT-001"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "T-shirt cotone S",
      "description": "T-shirt 100% cotone, manica corta, colore bianco",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true
    },
    {
      "box_id": 2,
      "name": "Scarpe sportive",
      "description": "Scarpe da corsa uomo",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9,
      "is_groupable": false
    }
  ],
  "manufacturer": {
    "name": "Fabbrica Esempio SRL",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Capannone 2",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  }
}
```

### Esempio di request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/quotation?level=detailed' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "USD",
    "address_sender": {"country_code": "BR", "city": "Diadema", "postal_code": "09941070"},
    "address_receiver": {"country_code": "US", "city": "Miami", "state_code": "FL", "postal_code": "33101"},
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "T-shirt cotone", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}]
  }'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Cotação Realizada com sucesso!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2026-02-21 14:28:10",
    "description": null,
    "channel": "Marketplace Esempio",
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_quote_exchange_value": "1",
    "currency_payment": "EUR",
    "currency_payment_exchange_value": "0.92",
    "warnings": [
      "Itens com descrições iguais podem causar problemas na fiscalização aduaneira"
    ],
    "carriers": [
      {
        "code": 1,
        "slug": "dhl-br",
        "name": "DHL",
        "url_image": "https://example.s3.amazonaws.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      }
    ]
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.quotation` | string (UUID) | **Codice della quotazione.** Conservalo: è la chiave di tutto il flusso |
| `data.finished` | boolean | Indica se l'elaborazione della quotazione è stata completata |
| `data.date` | string | Data/ora della quotazione |
| `data.channel` | string | Nome del canale di vendita, quando `channel_code` è stato indicato |
| `data.warnings` | array | **Opzionale.** Compare solo quando ci sono avvisi (es.: descrizioni duplicate, valori dichiarati elevati) |
| `data.carriers[]` | array | Opzioni di spedizione calcolate |
| `carriers[].code` | integer | Codice del contratto. Usalo in `carrier_code` nella spedizione |
| `carriers[].slug` | string | Identificatore testuale del contratto. Alternativa a `carrier_code` nella spedizione |
| `carriers[].name` | string | Nome commerciale del corriere |
| `carriers[].url_image` | string | Logo del corriere |
| `carriers[].valid_until` | string | **Validità della quotazione.** La spedizione deve essere creata prima di questa data/ora |
| `carriers[].transit_days` | integer | Tempo di transito stimato in giorni |
| `carriers[].weight_details.type` | string | `real` (peso reale) o `cubed` (peso volumetrico) — quale è stato usato nella tariffazione |
| `carriers[].weight_details.weight` | string | Peso considerato |
| `carriers[].freight` / `freight_final` | string | Nolo prima / dopo le spese amministrative |
| `carriers[].insurance` / `insurance_final` | string | Assicurazione prima / dopo le spese amministrative |
| `carriers[].tax` / `tax_final` | string | Imposte prima / dopo le spese amministrative |
| `carriers[].currency_quote_price` | string | Totale nella valuta della quotazione |
| `carriers[].currency_quote_amount` | string | Totale da pagare nella valuta della quotazione |
| `carriers[].currency_payment_amount` | string | Totale da pagare nella valuta di pagamento |

## Endpoint: consultare il risultato della quotazione

- **Metodo e rotta:** `GET /v1/quotation/{quotation}/{level}`
- **Quando usarlo:** per riconsultare un risultato già calcolato o cambiare il livello di dettaglio.

| Parametro | Posizione | Tipo | Obbligatorio | Valori | Descrizione |
|---|---|---|---|---|---|
| `quotation` | path | string | **sì** | UUID | Codice restituito alla creazione |
| `level` | path | string | no | `simple`, `detailed` | Predefinito `simple` |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

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

> Se il `quotation` non esiste o non appartiene al tuo account, l'API risponde `400` con
> `Não foi encontrada a cotação especificada!`.

## Regole di business

### Eleggibilità per la spedizione
Perché una quotazione possa generare una spedizione, tutte le condizioni seguenti devono essere vere:

1. Il `quotation` esiste e appartiene al tuo account.
2. La quotazione **non è scaduta** (rispetta il `valid_until` del corriere scelto).
3. La quotazione **non è stata ancora usata** in un'altra spedizione (`Cotação já foi utilizada em outra remessa!`).
4. La quotazione è stata creata nella **stessa versione** dell'API usata per la spedizione (v1 → v1).
5. Non è una quotazione `simple` di `package`/`pallet` (`Cotação Simples é inelegível para remessa!`).
6. Il corriere indicato è presente nell'elenco restituito dalla quotazione
   (`Contrato especificado não foi listado para esta cotação!`).

### Restrizioni di carico
La quotazione può essere rifiutata con `400` quando:

| Messaggio | Causa |
|---|---|
| `A caixa :name tem a altura/largura/profundidade maior do que a permitida!` | Dimensione oltre il limite dei contratti |
| `A caixa :name está com o peso maior que o permitido!` | Peso per collo oltre il limite |
| `Remessa está com o peso acima do permitido!` | Peso totale oltre il limite |
| `Nenhum item foi associado para a caixa :name !` | Scatola senza articoli in `advanced` |
| `A cotação excedeu o máximo de itens permitido!` | Numero di articoli oltre il limite |
| `Nenhum transportador encontrado para este usuário!` | Account senza contratti abilitati |
| `Não há contratos que atendem as especificações do pacote enviado...` | Nessun contratto copre la combinazione peso/dimensione/rotta |
| `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Carico fuori dalla fascia automatica — contatta il team commerciale |
| `Endereço de Remetente e Destinatário são iguais!` | Origine e destinazione identiche |
| `O país de origem é o mesmo de destino!` | L'API gestisce spedizioni internazionali |
| `Não foi passado um postal code para o endereço do Brasil no :name !` | Indirizzo brasiliano senza codice postale |
| `Este país de Origem/Destino está temporariamente inabilitado para envios!` | Rotta bloccata al momento |
| `Não existe tabela de duty para o país de destino :name com o hscode :hscode...` | HS Code senza tabella di imposta a destinazione — rivedi lo `hscode` |

### Registrazioni referenziate
Quando usi i codici delle registrazioni, questi devono esistere ed essere attivi:

| Campo | Registrazione | Pagina |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Scatola | [Scatola](api-caixa-v1.md) |
| `items[].sku` | Articolo | [Articoli](api-itens-v1.md) |
| `address_*.address_code` | Indirizzo | Registrato nel pannello |
| `channel_code` | Canale di vendita | Registrato nel pannello |
| `boxes[].kit` | Kit | Registrato nel pannello |

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Regola di business (restrizione di carico, rotta, contratto, HS Code) | Leggere `message` e adeguare lo scenario |
| `403` | Permesso di quotazione mancante | Richiedere il permesso al supporto |
| `406` | Struttura del payload incompatibile | Rivedere tipi e annidamento |
| `422` | Validazione dei campi | Correggere i campi elencati in `data` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- La risposta del `POST` contiene già il risultato calcolato — non è obbligatorio chiamare il `GET` subito dopo.
- `data.warnings` è informativo: la quotazione è stata eseguita, ma c'è un rischio doganale da rivedere.
- Per un payload di quotazione ancora più essenziale (solo paese e stato), vedi [Quotazione V2](api-cotacao-v2.md).
- Per quotazione + spedizione in un'unica chiamata sincrona, vedi [Manifesto](api-manifesto.md).
