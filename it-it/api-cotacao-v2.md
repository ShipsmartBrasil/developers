# Quotazione (v2)

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Stesso motore di calcolo della [Quotazione v1](api-cotacao-v1.md), con un **contratto di indirizzo molto più essenziale**.
- Usa la v2 quando, al momento della quotazione, **non hai** ancora città e codice postale del destinatario — solo il paese
  (e facoltativamente lo stato).
- Città, codice postale e indirizzo completo vengono indicati dopo, nella [Spedizione V2](api-remessa-v2.md).

> **Importante:** una quotazione creata in v2 può generare una spedizione solo in v2 (`POST /v2/shipping`).
> Vedi il [confronto](api-comparativo-cotacao-remessa.md) per scegliere la versione.

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
1. `POST /v2/quotation` indicando solo il paese (e lo stato, se lo conosci) in origine e a destinazione.
2. La risposta contiene già il risultato calcolato (`data.carriers`) e il codice `data.quotation`.
3. Scegli un corriere e crea la [Spedizione V2](api-remessa-v2.md) con l'indirizzo completo,
   **prima** del `valid_until`.

```
POST /v2/quotation  ──►  quotation + carriers[]
 (solo paese/stato)            │
                               ▼
                    POST /v2/shipping
           (indirizzo completo + carrier_code|slug)
```

## Differenze rispetto alla v1

| Tema | v1 | v2 |
|---|---|---|
| Indirizzo nella quotazione | `country_code` + `city` + (`postal_code` o `state_code`/`state`) | Solo `country_code` (+ `state_code` opzionale) |
| Indirizzo nella spedizione | Indirizzo, nome, e‑mail, telefono, tipo | Indirizzo, nome, e‑mail, telefono, tipo **+ città, stato e codice postale** |
| Consultazione del risultato | `GET /v1/quotation/{quotation}/{level}` | **Non esiste.** Il risultato arriva nel `POST` stesso |
| `ior` (Importer of Record) | Supportato | **Non fa parte del contratto della v2** |
| `items[].selling_unit_value` | Supportato | **Non fa parte del contratto della v2** |
| Precisione del calcolo | Maggiore: città/codice postale permettono di applicare tasse regionali già nella quotazione | Il valore può essere affinato nella spedizione, quando viene indicato l'indirizzo completo |

> Tutto il resto (oggetto, tipo, scatole, articoli, kit, valute, nolo dichiarato, produttore, warning)
> è identico alla v1.

## Concetti essenziali

### `object` — cosa viene spedito
| Valore | Significato | Note |
|---|---|---|
| `doc` | Documento | Quotazione per scatola, indicando `weight` e `price` |
| `package` | Pacco/scatola | Flusso più comune |
| `pallet` | Pallet | Permette di indicare `volume` invece di altezza/larghezza/profondità |
| `not_doc` | Legacy | Convertito internamente in `package`. **Non usarlo nelle nuove integrazioni** |

### `type` — come viene costruito il calcolo
| Valore | Cosa invii | Quando usarlo |
|---|---|---|
| `simple` | Solo `boxes[]` con `weight` e `price` | Simulazione rapida. **Non è eleggibile per la spedizione quando `object` è `package` o `pallet`** |
| `advanced` | `boxes[]` (dimensioni) + `items[]` con `box_id` | Flusso standard per la spedizione |
| `items` | Solo `items[]` con `sku` | Quando articoli e scatole sono già registrati e la piattaforma fa l'incastro |

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

Inviato come **query string**: `POST /v2/quotation?level=detailed`.

## Endpoint: creare una quotazione

- **Metodo e rotta:** `POST /v2/quotation`
- **Query string opzionale:** `level=simple|detailed`

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
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Consegna a un indirizzo residenziale |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Colli non impilabili |
| `description` | string | no | - | 155 | `null` | - | Descrizione libera della quotazione |
| `channel_code` | integer | no | - | - | preset dell'account | - | Codice del canale di vendita registrato |
| `channel_identifier_number` | string | no | - | - | `null` | - | Il tuo identificatore commerciale (n° dell'ordine). **Consigliato** |
| `channel_order_number` | string | no | - | 100 | `null` | - | Numero dell'ordine nel canale di vendita |
| `declared_freight` | object | no | - | - | preset dell'account | - | Nolo dichiarato (vedi tabella sotto) |
| `manufacturer` | object | no | - | - | `null` | - | Produttore mostrato sulla Commercial Invoice (vedi tabella sotto) |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Livello di dettaglio del risultato |

### Parametri — `address_sender` e `address_receiver`

**Questa è la differenza principale della v2.** Scegli uno dei due modi:

**Modo A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questo modo | Codice di un indirizzo registrato sul tuo account |

**Modo B — solo il paese (e lo stato, se lo conosci):**

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `country_code` | string | **sì** | 2 | 2 | Paese in ISO 3166-1 alpha-2 (es.: `BR`, `US`) |
| `state_code` | string | no | - | - | Sigla dello stato/provincia (es.: `SP`, `FL`). Quando indicata, migliora la precisione del calcolo delle imposte regionali |

> Non inviare `city`, `postal_code` né `state` nella quotazione v2 — questi dati vengono indicati nella spedizione.

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

### Parametri — `items[]`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `quantity` | integer | **sì** | 1 | 1000 | Quantità dell'articolo |
| `unit_value` | number | **sì** | - | - | Valore unitario nella valuta della quotazione |
| `box_id` | integer | **sì**, tranne quando `type` è `items` | - | - | **Posizione** della scatola nell'array `boxes` (1 = prima scatola). Non è il `box_code` |
| `sku` | string | **sì** quando `type` è `items` | - | 50 | SKU dell'articolo registrato ([Articoli](api-itens-v1.md)). Quando indicato, vengono usati i dati della registrazione |
| `ean` | string | no | - | 50 | Codice EAN dell'articolo |
| `name` | string | **sì** quando `sku` non è indicato | 6 | 50 | Nome dell'articolo (compare sull'invoice) |
| `description` | string | no | - | 155 | Descrizione dettagliata dell'articolo |
| `country_code` | string | **sì** quando `sku` non è indicato | 2 | 2 | Paese di origine/fabbricazione dell'articolo |
| `hscode` | string | **sì** quando `sku` non è indicato | 6 | 10 | NCM/HS Code. Definisce la tabella delle imposte applicata |
| `weight` | number | **sì** quando `sku` non è indicato | 0.001 | - | Peso unitario dell'articolo |
| `is_groupable` | boolean | no | - | - | Predefinito `true`. Quando è `false`, l'articolo riceve una scatola esclusiva nel raggruppamento `smartbox` |

### Parametri — `declared_freight`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valore del nolo dichiarato |
| `currency` | string | **sì** quando `value` è indicato | 3 | 3 | Valuta del nolo dichiarato |

### Parametri — `manufacturer`

Dati del produttore stampati sulla Commercial Invoice. Invia `address_code` **oppure** i campi manuali.

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `address_code` | integer | alternativa ai campi manuali | - | - | Codice di un indirizzo registrato |
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

### Payload minimo

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
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

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "channel_order_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
  "residential_delivery": false,
  "non_stackable": false,
  "description": "Ordine del marketplace, lotto di ottobre",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
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
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
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
      "is_groupable": true
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
  --url 'https://api.shipsmart.global/v2/quotation?level=detailed' \
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
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
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
        "weight_details": { "type": "real", "weight": "5" },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "insurance": "11.55",
        "insurance_final": "11.55",
        "tax": "24924.09",
        "tax_final": "24924.09",
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.quotation` | string (UUID) | **Codice della quotazione.** Usalo in `POST /v2/shipping` |
| `data.finished` | boolean | Indica se l'elaborazione della quotazione è stata completata |
| `data.warnings` | array | **Opzionale.** Compare solo quando ci sono avvisi doganali |
| `carriers[].code` | integer | Codice del contratto. Usalo in `carrier_code` nella spedizione |
| `carriers[].slug` | string | Identificatore testuale del contratto. Alternativa a `carrier_code` |
| `carriers[].valid_until` | string | **Validità della quotazione.** Crea la spedizione prima di questa data/ora |
| `carriers[].transit_days` | integer | Tempo di transito stimato in giorni |
| `carriers[].weight_details.type` | string | `real` o `cubed` — quale peso è stato usato nella tariffazione |
| `carriers[].freight` / `freight_final` | string | Nolo prima / dopo le spese amministrative |
| `carriers[].insurance` / `insurance_final` | string | Assicurazione prima / dopo le spese amministrative |
| `carriers[].tax` / `tax_final` | string | Imposte prima / dopo le spese amministrative |
| `carriers[].currency_payment_amount` | string | Totale da pagare nella valuta di pagamento |

## Regole di business

### Eleggibilità per la spedizione
1. Il `quotation` esiste e appartiene al tuo account.
2. La quotazione **non è scaduta** (rispetta il `valid_until`).
3. La quotazione **non è stata ancora usata** in un'altra spedizione.
4. La quotazione è stata creata nella **v2** e verrà usata in `POST /v2/shipping`.
5. Non è una quotazione `simple` di `package`/`pallet`.
6. Il corriere indicato è presente nell'elenco restituito dalla quotazione.

### Restrizioni di carico
Valgono le stesse restrizioni della v1 (dimensioni, peso per collo, peso totale, limite di articoli,
disponibilità del contratto, rotta bloccata, HS Code senza tabella di imposta).
Consulta la sezione **Restrizioni di carico** in [Quotazione (v1)](api-cotacao-v1.md#restrizioni-di-carico).

### Nessun endpoint di consultazione
La v2 **non dispone** di `GET /v2/quotation/{quotation}`. Il risultato completo viene consegnato nella risposta del `POST`,
quindi **memorizza la risposta** se hai bisogno di ripresentarla all'utente.

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Regola di business (restrizione di carico, rotta, contratto, HS Code) | Leggere `message` e adeguare lo scenario |
| `403` | Permesso di quotazione mancante | Richiedere il permesso al supporto |
| `406` | Struttura del payload incompatibile | Rivedere tipi e annidamento |
| `422` | Validazione dei campi | Correggere i campi elencati in `data` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- La v2 scambia precisione con semplicità: l'indirizzo completo entra solo nella spedizione.
- I flussi operativi di **status, documenti, annullamento e pianificazione del ritiro** usano gli stessi
  endpoint `/v1/...`, anche per le spedizioni create in v2 — vedi
  [Tracciamento](api-rastreio.md), [Documenti](api-documentos.md) e [Pianificazione](api-agendamento.md).
- Conserva la risposta del `POST`: nella v2 non esiste un endpoint di riconsultazione.
