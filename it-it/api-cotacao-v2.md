# Quotazione (v2)

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Stesso calcolo della [Quotazione v1](api-cotacao-v1.md), con un **contratto di indirizzo molto più essenziale**.
- Usa la v2 quando, al momento della quotazione, **non hai ancora** città e CAP — solo il paese
  (e facoltativamente lo stato).
- La città, il CAP e l'indirizzo completo vengono indicati e validati in seguito, nella [Spedizione V2](api-remessa-v2.md).
- Copre gli scenari di carico della v1 **e** la **quotazione per articoli** (`type: items`), **esclusiva della v2**: invii
  solo lo `sku` degli articoli registrati, senza indicare le scatole — vedi [Scenari di quotazione](#scenari-di-quotazione).

> **Importante:** una quotazione creata nella v2 può generare una spedizione solo nella v2 (`POST /v2/shipping`).
> Consulta il [confronto](api-comparativo-cotacao-remessa.md) per scegliere la versione.

## URL di base
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
1. Scegli lo [scenario di quotazione](#scenari-di-quotazione) che corrisponde al tuo carico (`object` + `type`).
2. `POST /v2/quotation` indicando solo il paese (e lo stato, se lo conosci) all'origine e alla destinazione.
3. La risposta contiene già il risultato calcolato (`data.carriers`) e il codice `data.quotation`. **Conserva la risposta**:
   nella v2 non esiste un endpoint di riconsultazione.
4. Verifica che la quotazione sia [idonea alla spedizione](#quotazioni-non-idonee-alla-spedizione).
5. Scegli un corriere e crea la [Spedizione V2](api-remessa-v2.md) con l'indirizzo completo,
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
| Quotazione per articoli (`type: items`) | **Non disponibile**: le scatole vengono sempre indicate | **Disponibile**: solo lo `sku` degli articoli registrati, senza scatole |
| Validazione dell'indirizzo | Nella quotazione (CAP, città, stato) | Nella spedizione, quando viene inviato l'indirizzo completo |
| Indirizzo nella spedizione | Via, nome, e‑mail, telefono, tipo | Via, nome, e‑mail, telefono, tipo **+ città, stato e CAP** |
| Consultazione del risultato | `GET /v1/quotation/{quotation}/{level}` | **Non esiste.** Il risultato arriva nello stesso `POST` |
| `ior` (Importer of Record) | Supportato | **Non fa parte del contratto della v2** (ignorato) |
| `items[].selling_unit_value` | Supportato | **Non fa parte del contratto della v2** (ignorato) |
| Precisione del calcolo | Maggiore: città/CAP permettono di applicare tariffe regionali già nella quotazione | Il valore considera paese/stato; l'indirizzo completo entra nella spedizione |

> Queste sono le due ragioni di esistere della v2: **indirizzo ridotto** (solo paese/stato) e **quotazione per articoli**.
> Tutto il resto (oggetto, scatole, kit, valute, nolo dichiarato, produttore, warning, preimpostazioni dell'account) è identico alla v1.

## Concetti essenziali

### `object` — cosa viene spedito
| Valore | Significato | Osservazioni |
|---|---|---|
| `doc` | Documento | Quotato per collo, indicando `weight` e `price` di ogni busta/scatola |
| `package` | Pacco/scatola | Flusso più comune |
| `pallet` | Pallet | Accetta dimensioni (LLA) **o** `volume` in m³/ft³ |
| `not_doc` | Legacy | Convertito internamente in `package`. **Non usarlo nelle nuove integrazioni** |

### `type` — come viene impostato il calcolo
| Valore | Cosa invii | Quando usarlo |
|---|---|---|
| `simple` | Solo `boxes[]` con `weight` e `price` | Documenti e simulazioni rapide. **Non è idoneo alla spedizione quando `object` è `package` o `pallet`** |
| `advanced` | `boxes[]` (dimensioni) + `items[]` con `box_id` | Flusso standard per la spedizione |
| `items` | `items[]` con `sku` (e, in base al raggruppamento dell'account, `boxes[]` candidate) | **Esclusivo della v2.** Quando gli articoli sono già registrati e la piattaforma compone le scatole |

### Matrice di idoneità

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Quotazione e spedizione | Non consigliato | Non consigliato |
| `package` | ⚠️ Solo simulazione (non idonea alla spedizione) | ✅ Quotazione e spedizione | ✅ Quotazione e spedizione |
| `pallet` | ⚠️ Solo simulazione (non idonea alla spedizione) | ✅ Quotazione e spedizione | ✅ Quotazione e spedizione |

### `tax` — chi paga le imposte
| Valore | Significato |
|---|---|
| `sender` | DDP: il mittente paga imposte e tasse a destinazione |
| `receiver` | DDU: il destinatario paga imposte e tasse a destinazione |

> Se il tuo account ha una **preimpostazione dell'imposta** configurata, essa **sostituisce** il valore inviato in `tax`.
> Vedi [Preimpostazioni applicate nella quotazione](api-cotacao-v1.md#preimpostazioni-applicate-nella-quotazione).

### `measurement` — sistema di misura
| Valore | Dimensioni | Peso | Volume |
|---|---|---|---|
| `metric` (predefinito) | centimetri | chilogrammi | m³ |
| `imperial` | pollici | libbre | ft³ |

### `level` — livello di dettaglio del risultato
| Valore | Risultato |
|---|---|
| `simple` (predefinito) | Elenco essenziale dei corrieri con i totali |
| `detailed` | Include il cambio e la composizione dettagliata di nolo, assicurazione e imposta, con le regole di sussidio applicate |

Inviato come **query string**: `POST /v2/quotation?level=detailed`.

## Endpoint: creare la quotazione

- **Metodo e rotta:** `POST /v2/quotation`
- **Query string opzionale:** `level=simple|detailed`
- **Quando usarlo:** quando, al momento della quotazione, conosci solo il paese (e forse lo stato) di origine e destinazione.

### Parametri — radice del payload

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Valori accettati | Descrizione |
|---|---|---|---|---|---|---|---|
| `object` | string | **sì** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | Cosa viene spedito |
| `type` | string | **sì** | - | - | - | `simple`, `advanced`, `items` | Come viene impostato il calcolo (`items` esiste solo nella v2) |
| `tax` | string | **sì** | - | - | - | `sender`, `receiver` | Chi paga le imposte (può essere sostituito dalla preimpostazione dell'account) |
| `insurance` | boolean | **sì** | - | - | - | `true`, `false` | Attivare l'assicurazione (può essere attivata automaticamente dall'account) |
| `currency_quote` | string | **sì** | 3 | 3 | - | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta in cui viene presentata la quotazione e in cui vengono indicati `unit_value`/`price` |
| `currency_payment` | string | **sì** | 3 | 3 | - | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta in cui verrà effettuato il pagamento |
| `address_sender` | object | **sì** | - | - | - | - | Origine (vedi tabella sotto) |
| `address_receiver` | object | **sì** | - | - | - | - | Destinazione (vedi tabella sotto) |
| `boxes` | array | condizionale | - | - | - | - | Obbligatorio quando `type` **non** è `items`. In `type: items` con raggruppamento `smartbox`, è anch'esso obbligatorio |
| `items` | array | condizionale | - | - | - | - | Obbligatorio quando `type` **non** è `simple` e nessuna scatola usa `kit` |
| `measurement` | string | no | - | - | preset dell'account o `metric` | `metric`, `imperial` | Sistema di misura |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Consegna a un indirizzo residenziale |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Colli non impilabili |
| `description` | string | no | - | 155 | `null` | - | Descrizione libera della quotazione |
| `channel_code` | integer | no | - | - | preset dell'account | - | Codice del canale di vendita registrato e abilitato per il tuo utente |
| `channel_identifier_number` | string | no | - | - | `null` | - | Il tuo identificatore commerciale (n. dell'ordine). **Consigliato** |
| `channel_order_number` | string | no | - | 100 | `null` | - | Numero dell'ordine nel canale di vendita |
| `declared_freight` | object | no | - | - | preset dell'account | - | Nolo dichiarato. Richiede un permesso specifico |
| `manufacturer` | object | no | - | - | `null` | - | Produttore mostrato nella Commercial Invoice |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Livello di dettaglio del risultato |

### Parametri — `address_sender` e `address_receiver`

**Questa è la differenza principale della v2.** Scegli uno dei due modi e non mescolarli:

**Modo A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questo modo | Codice di un indirizzo registrato nel pannello del tuo account |

**Modo B — solo il paese (e lo stato, se lo conosci):**

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `country_code` | string | **sì** | 2 | 2 | Paese in ISO 3166-1 alpha-2 (es.: `BR`, `US`) |
| `state_code` | string | no | - | - | Sigla dello stato/provincia (es.: `SP`, `FL`). Quando indicato, migliora la precisione del calcolo delle imposte regionali e viene riutilizzato nella spedizione |

> Non inviare `city`, `postal_code` né `state` nella quotazione v2: questi campi vengono **ignorati** e devono essere
> indicati nella [Spedizione V2](api-remessa-v2.md).

> **Mittente registrato obbligatorio per alcuni tipi di account:** gli account di persona fisica e gli account aziendali
> con un unico centro di distribuzione devono indicare il mittente con il **Modo A** (`address_code`).
> Senza di esso, l'API risponde `406` con `Codigo de endereço do remetente não encontrado!`.

### Parametri — `boxes[]`

Identico alla v1. Ci sono **tre modi** per descrivere il collo:

**Modo A — kit registrato** (compila automaticamente scatola **e** articoli):

| Campo | Tipo | Obbligatorio | Max. | Descrizione |
|---|---|---|---|---|
| `kit` | string | **sì** in questo modo | 50 | SKU del kit registrato. La scatola e gli articoli di quella posizione vengono compilati dalla registrazione del kit |

**Modo B — scatola registrata:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `box_code` | integer | **sì** in questo modo | Codice della scatola registrata ([Scatola](api-caixa-v1.md)) |
| `box_identifier` | string | alternativa a `box_code` | Identificatore della scatola registrata (es.: `CX0236`) |

**Modo C — scatola indicata manualmente:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `name` | string | no | 4 | 60 | Nome/etichetta del collo |
| `height` | number | **sì** (eccetto pallet con `volume`) | 1 | - | Altezza |
| `width` | number | **sì** (eccetto pallet con `volume`) | 1 | - | Larghezza |
| `depth` | number | **sì** (eccetto pallet con `volume`) | 1 | - | Profondità (lunghezza) |
| `volume` | number | condizionale | 0.001 | - | **Solo quando `object` è `pallet`.** Alternativa ad altezza/larghezza/profondità, in m³ o ft³ |

**Campi validi nei modi B e C:**

| Campo | Tipo | Obbligatorio | Min. | Descrizione |
|---|---|---|---|---|
| `weight` | number | **sì** quando `object` è `doc`/`package` **e** `type` è `simple` | > 0 | Peso del collo. In `advanced`/`items`, se omesso, è la somma degli articoli |
| `price` | number | **sì** quando `object` è `doc`/`package` **e** `type` è `simple` | > 0 | Valore dichiarato del collo. In `advanced`/`items`, se omesso, è la somma degli articoli |

### Parametri — `items[]`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `quantity` | integer | **sì** | 1 | 1000 | Quantità dell'articolo |
| `unit_value` | number | **sì** | > 0 | - | Valore unitario nella valuta della quotazione. Zero o negativo viene rifiutato |
| `box_id` | integer | **sì**, eccetto quando `type` è `items` | 1 | n. di scatole | **Posizione** della scatola nell'array `boxes` (1 = prima scatola). Non è il `box_code` |
| `sku` | string | **sì** quando `type` è `items` | - | 50 | SKU dell'articolo registrato ([Articoli](api-itens-v1.md)). I campi omessi vengono completati dalla registrazione |
| `ean` | string | no | - | 50 | Codice EAN dell'articolo |
| `nsmp` | string | no | - | 50 | *Non Standard Manufacturer Product Id*: identificatore del prodotto presso il produttore. Stampato sulla invoice e usato nelle informazioni regolatorie richieste da alcuni corrieri per destinazioni nell'Unione Europea |
| `name` | string | **sì** quando `sku` non viene indicato | 6 | 50 | Nome dell'articolo (compare sulla invoice) |
| `description` | string | no | - | 155 | Descrizione dettagliata dell'articolo |
| `country_code` | string | **sì** quando `sku` non viene indicato | 2 | 2 | Paese di origine/fabbricazione dell'articolo |
| `hscode` | string | **sì** quando `sku` non viene indicato | 6 | 10 | NCM/HS Code. Definisce la tabella delle imposte applicata |
| `weight` | number | **sì** quando `sku` non viene indicato | 0.001 | - | Peso unitario dell'articolo |
| `is_groupable` | boolean | no | - | - | Predefinito `true`. Quando `false`, l'articolo riceve una scatola esclusiva nei raggruppamenti di `type: items` |

### Parametri — `declared_freight`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valore del nolo dichiarato |
| `currency` | string | **sì** quando `value` viene indicato | 3 | 3 | Valuta del nolo dichiarato |

> Applicato solo se il tuo profilo ha il **permesso di nolo dichiarato**; altrimenti viene ignorato silenziosamente.

### Parametri — `manufacturer`

Identico alla v1: invia `address_code` **o** i campi manuali (`name`, `country_code`, `state_code`, `state`,
`postal_code`, `city`, `street`, `number`, `district`, `complement`, `email`, `phone`, `phone_extension`, `type`,
`federal_tax_id`, `state_tax_id`). Limiti e raccomandazioni nella
[tabella del produttore della v1](api-cotacao-v1.md#parametri--manufacturer).

| Campo | Tipo | Consigliato | Descrizione |
|---|---|---|---|
| `address_code` | integer | alternativa ai campi manuali | Indirizzo registrato del produttore |
| `name` | string | **sì** senza `address_code` | Ragione sociale/nome del produttore (da 5 a 30 caratteri) |

### Payload minimo

Il payload più piccolo accettato dall'endpoint (quotazione di documento):

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    {
      "height": 30,
      "width": 25,
      "depth": 2,
      "weight": 0.5,
      "price": 10
    }
  ]
}
```

### Payload massimo

Tutti i campi accettati dall'endpoint, con i tre modi di indicare la scatola:

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
    { "name": "Scatola master", "height": 20, "width": 20, "depth": 20, "weight": 2.5, "price": 150 },
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "T-shirt cotone S",
      "description": "T-shirt 100% cotone, manica corta, colore bianco",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
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
      "is_groupable": false
    },
    {
      "box_id": 3,
      "sku": "CNC-010-M",
      "quantity": 2,
      "unit_value": 15
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
        "url_image": "https://files.example.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "real", "weight": "5" },
        "freight": "181.95",
        "freight_final": "181.95",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.45",
        "insurance_final": "11.45",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "28153.894",
        "tax_final": "28153.894",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "url_image": "https://files.example.com/contratos/fedex.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.55",
        "insurance_final": "11.55",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "24924.09",
        "tax_final": "24924.09",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Campi della risposta

Stessa struttura della v1 — descrizione completa in [Campi della risposta della v1](api-cotacao-v1.md#campi-della-risposta).
I campi più usati:

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.quotation` | string (UUID) | **Codice della quotazione.** Usalo in `POST /v2/shipping` |
| `data.finished` | boolean | `true` quando la quotazione **è già stata usata** per creare una spedizione |
| `data.warnings` | array | **Opzionale.** Compare solo quando ci sono avvisi doganali |
| `carriers[].code` | integer | Codice del contratto. Usalo in `carrier_code` nella spedizione |
| `carriers[].slug` | string | Identificatore testuale del contratto. Alternativa a `carrier_code` |
| `carriers[].valid_until` | string | **Validità della quotazione.** Crea la spedizione prima di questa data/ora |
| `carriers[].transit_days` | integer | Tempo di transito stimato in giorni |
| `carriers[].weight_details.type` | string | `real` o `cubed` — quale peso è stato usato nella tariffazione |
| `carriers[].freight` / `freight_final` | string | Nolo prima / dopo le commissioni amministrative |
| `carriers[].insurance` / `insurance_final` | string | Assicurazione prima / dopo le commissioni amministrative |
| `carriers[].tax` / `tax_final` | string | Imposte prima / dopo le commissioni amministrative |
| `carriers[].currency_quote_amount` | string | Totale da pagare nella valuta della quotazione |
| `carriers[].currency_payment_amount` | string | Totale da pagare nella valuta del pagamento |

## Scenari di quotazione

Ogni scenario seguente riporta cosa cambia nel payload, il payload minimo e il payload massimo **di quello scenario**.
I campi comuni della radice (valute, imposta, assicurazione, canale, nolo dichiarato ecc.) seguono la
[tabella della radice del payload](#parametri--radice-del-payload).

| Scenario | `object` | `type` | `boxes[]` | `items[]` | Idonea alla spedizione |
|---|---|---|---|---|---|
| [Documento](#quotazione-di-documento) | `doc` | `simple` | Dimensioni + `weight` + `price` | Non usato | ✅ |
| [Pacco — semplice](#quotazione-di-pacco-scatola--semplice) | `package` | `simple` | Dimensioni + `weight` + `price` | Non usato | ❌ solo simulazione |
| [Pacco — avanzato](#quotazione-di-pacco-scatola--avanzata) | `package` | `advanced` | Dimensioni (peso/valore opzionali) | Obbligatorio, con `box_id` | ✅ |
| [Pallet — per LLA](#quotazione-di-pallet--per-lla) | `pallet` | `advanced` | `height` + `width` + `depth` | Obbligatorio, con `box_id` | ✅ |
| [Pallet — per m³](#quotazione-di-pallet--per-m³) | `pallet` | `advanced` | `volume` | Obbligatorio, con `box_id` | ✅ |
| [Articoli](#quotazione-di-articoli) — **esclusivo della v2** | `package` | `items` | Dipende dal raggruppamento dell'account | Obbligatorio, con `sku` | ✅ |

### Quotazione di Documento

- **Quando usarla:** invio di carte/documenti senza valore commerciale rilevante (contratti, certificati, stampati).
- **Come comporla:** `object: doc` + `type: simple`. Ogni collo (busta/scatola) indica dimensioni, `weight` e `price`.
- **Non usa** `items[]` né `manufacturer`: i documenti non passano per il calcolo delle imposte per articolo.
- **È idonea alla spedizione.**

#### Payload minimo

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 }
  ]
}
```

#### Payload massimo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0002",
  "channel_order_number": "334456",
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Invio di contratti firmati",
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "declared_freight": { "value": 20, "currency": "BRL" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Busta A4", "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 },
    { "box_code": 7, "weight": 1.2, "price": 15 }
  ]
}
```

> In `doc`, `weight` e `price` sono obbligatori **in ogni collo**, anche quando la scatola è registrata (`box_code`/`box_identifier`).

### Quotazione di Pacco (Scatola) — semplice

- **Quando usarla:** simulazione rapida del nolo quando conosci solo il peso e il valore di ogni scatola.
- **Come comporla:** `object: package` + `type: simple`. Ogni scatola indica dimensioni, `weight` e `price`. Senza articoli.
- ⚠️ **Non è idonea alla spedizione.** Per spedire effettivamente, rifai la quotazione come
  [Pacco — avanzato](#quotazione-di-pacco-scatola--avanzata) o [Articoli](#quotazione-di-articoli).

#### Payload minimo

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 }
  ]
}
```

#### Payload massimo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "SIM-2026-0001",
  "channel_order_number": "334457",
  "object": "package",
  "type": "simple",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": true,
  "description": "Simulazione del nolo per la vetrina",
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "declared_freight": { "value": 30, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Scatola media", "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 },
    { "box_identifier": "CX0236", "weight": 3.5, "price": 180 },
    { "box_code": 3, "weight": 1, "price": 60 },
    { "kit": "KIT-001" }
  ]
}
```

> Nella scatola del `kit`, il peso e le dimensioni provengono dalla registrazione della scatola del kit e il `price` è la somma dei valori degli articoli del kit.
> Se la scatola registrata ha un **peso massimo** configurato, il `weight` indicato non può superarlo.

### Quotazione di Pacco (Scatola) — avanzata

- **Quando usarla:** è il **flusso standard per la spedizione** di merci. Dichiari gli articoli e indichi in quale scatola
  si trova ciascuno.
- **Come comporla:** `object: package` + `type: advanced`. Le scatole riportano le dimensioni; ogni articolo indica la
  posizione della propria scatola in `box_id` (1 = prima scatola).
- Il peso e il valore di ogni scatola vengono calcolati come somma degli articoli, se non indicati.
- **Ogni scatola deve avere almeno un articolo**; un `box_id` fuori dall'intervallo delle scatole viene rifiutato.
- **È idonea alla spedizione.**

#### Payload minimo

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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

#### Payload massimo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0003",
  "channel_order_number": "334458",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Ordine con due scatole",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Scatola abbigliamento", "height": 30, "width": 25, "depth": 15, "weight": 1.2, "price": 120 },
    { "box_code": 3 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "T-shirt cotone S",
      "description": "T-shirt 100% cotone, manica corta, colore bianco",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 4,
      "unit_value": 29.9,
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
      "unit_value": 199.9
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

> Un articolo con `sku` registrato può essere inviato solo con `box_id`, `sku`, `quantity` e `unit_value`: il resto
> (nome, paese, HS Code, peso, EAN, `nsmp`) viene completato dalla [registrazione degli articoli](api-itens-v1.md).
> Ciò che invii nel payload ha priorità sulla registrazione.

### Quotazione di Pallet — per LLA

- **Quando usarla:** carico pallettizzato di cui conosci le misure — **L**unghezza (`depth`), **L**arghezza (`width`)
  e **A**ltezza (`height`).
- **Come comporla:** `object: pallet` + `type: advanced`, con `height`, `width` e `depth` in ogni pallet e gli articoli
  che indicano il pallet in `box_id`.
- La piattaforma calcola il volume (`L × L × A ÷ 1.000.000` = m³) a partire dalle misure.
- Il peso del pallet, se omesso, è la somma dei pesi degli articoli. Indica `weight` per includere il peso della base/imballaggio.
- `type: simple` è accettato per la simulazione, ma è **non idoneo alla spedizione**.

#### Payload minimo

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 150, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Ricambio industriale",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Payload massimo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0001",
  "channel_order_number": "990011",
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "description": "Due pallet di ricambi industriali",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Pallet 01", "height": 150, "width": 100, "depth": 120, "weight": 280, "price": 900 },
    { "name": "Pallet 02", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Ricambio industriale",
      "description": "Ingranaggio in acciaio per macchina confezionatrice",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Cuscinetto industriale",
      "description": "Cuscinetto a sfere in acciaio",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

### Quotazione di Pallet — per m³

- **Quando usarla:** carico pallettizzato di cui conosci solo il **volume** (cubatura), senza le misure singole.
- **Come comporla:** `object: pallet` + `type: advanced`, con `volume` in ogni pallet (m³ in `metric`, ft³ in `imperial`).
- La piattaforma converte il volume in un cubo equivalente per la tariffazione:
  `lato (cm) = ∛volume (m³) × 100`. Es.: `1.0` m³ → 100 × 100 × 100 cm; `1.8` m³ → 121,64 × 121,64 × 121,64 cm.
- Se invii `volume` **e** le tre misure, le misure (LLA) hanno la priorità.
- Puoi combinare pallet per LLA e per m³ nello stesso payload (ogni posizione di `boxes[]` decide il proprio modo).

#### Payload minimo

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "volume": 1.8 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Ricambio industriale",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Payload massimo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0002",
  "channel_order_number": "990012",
  "object": "pallet",
  "type": "advanced",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "description": "Pallet per cubatura e pallet per misure",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Pallet cubatura", "volume": 1.8, "weight": 300, "price": 900 },
    { "name": "Pallet misurato", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Ricambio industriale",
      "description": "Ingranaggio in acciaio per macchina confezionatrice",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Cuscinetto industriale",
      "description": "Cuscinetto a sfere in acciaio",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> Un pallet senza misure **e** senza `volume` viene rifiutato con `422` (validazione) o con
> `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³`.

### Quotazione di Articoli

- **Esclusiva della v2.** È uno dei motivi per scegliere la v2: il payload non deve descrivere il pacco (scatola).
- **Quando usarla:** hai già i prodotti nella [registrazione degli articoli](api-itens-v1.md) e vuoi che la piattaforma
  componga le scatole da sola.
- **Come comporla:** `object: package` + `type: items`. Ogni articolo indica `sku`, `quantity` e `unit_value`; **non**
  indicare `box_id`.
- Il modo in cui vengono composte le scatole dipende dal **raggruppamento** configurato nel tuo account
  ([Primi Passi](api-primeiros-passos.md#raggruppamento-delle-scatole)):

| Raggruppamento | `boxes[]` nel payload | Prerequisito nella registrazione dell'articolo | Come vengono composte le scatole |
|---|---|---|---|
| `default` | Non usato | Articolo collegato a una scatola registrata (`box_code` o `box_identifier`) | Gli articoli raggruppabili che usano la stessa scatola vengono riuniti; gli articoli non raggruppabili ricevono una scatola per unità |
| `all_in_one` | Non usato | Articolo con dimensioni registrate | Gli articoli raggruppabili finiscono in **una** scatola virtuale con il volume totale; i non raggruppabili ricevono una scatola per unità |
| `smartbox` | **Obbligatorio** (scatole candidate) | Articolo con dimensioni registrate | Gli articoli vengono inseriti nelle scatole inviate, considerando volume e dimensioni con un margine del 10% e `is_groupable`/`is_foldable` |

#### Payload minimo

Raggruppamento `default` o `all_in_one`:

```json
{
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "items": [
    { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
  ]
}
```

#### Payload massimo

Raggruppamento `smartbox` (scatole candidate indicate):

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0004",
  "channel_order_number": "334459",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Ordine composto dalla registrazione degli articoli",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 25, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "name": "Scatola grande", "height": 40, "width": 40, "depth": 40 }
  ],
  "items": [
    {
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "T-shirt cotone S",
      "description": "T-shirt 100% cotone, manica corta, colore bianco",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "sku": "TEN-042-M",
      "quantity": 1,
      "unit_value": 199.9,
      "is_groupable": false
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> Errori tipici di questo scenario: `O item com o sku :name não existe!` (`404`),
> `A caixa com o box_code/box_identifier :name não existe ou está desativada` (`404`),
> `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (`400`) e
> `Nenhuma caixa comporta o item :name (Smart Box).` (`400`).

### Quotazione con Kit

Un **kit** è un insieme preregistrato nel pannello (una scatola + articoli con quantità). Indicando
`boxes[].kit`, quella posizione viene sostituita dalla scatola del kit e gli articoli del kit vengono inclusi automaticamente.

- Può essere combinato con scatole comuni nello stesso payload.
- Se **tutte** le scatole sono kit, `items[]` può essere omesso.
- In `type: simple`, il kit diventa una scatola con `price` = somma degli articoli del kit; in `advanced`, gli articoli del kit
  entrano nella quotazione indicando la posizione del kit.

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "kit": "KIT-001" }
  ]
}
```

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `O kit :sku é inválido!` | Kit inesistente, inattivo o di un altro account |
| `400` | `A caixa do kit :sku é inválida!` | La scatola collegata al kit non esiste o è stata disattivata |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Qualche articolo del kit è stato rimosso dalla registrazione |

## Quotazioni non idonee alla spedizione

Le stesse condizioni della v1 valgono per la v2, verificate nel `POST /v2/shipping` in quest'ordine:

| # | Condizione | Status | Messaggio in caso di errore |
|---|---|---|---|
| 1 | La quotazione esiste ed è stata creata nella **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | La quotazione appartiene al tuo account | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Account con imposta vincolata a DDP: quotazione DDU (`tax: receiver`) senza pagamento approvato | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | La quotazione **non è ancora stata usata** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | Non è `type: simple` con `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | La quotazione **non è scaduta** (`valid_until` del corriere) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | Il corriere scelto è presente in `carriers[]` di questa quotazione | `404` | `Contrato especificado não foi listado para esta cotação!` |

> Nella v2 l'indirizzo completo viene validato solo nella spedizione. Un CAP o un indirizzo non valido indicato nella
> [Spedizione V2](api-remessa-v2.md#validazione-dellindirizzo-completo) impedisce anch'esso la creazione.

## Regole di business

### Preimpostazioni e restrizioni di carico
Valgono le stesse regole della v1:
[Preimpostazioni applicate nella quotazione](api-cotacao-v1.md#preimpostazioni-applicate-nella-quotazione),
[Restrizioni di carico](api-cotacao-v1.md#restrizioni-di-carico),
[Avvisi della quotazione](api-cotacao-v1.md#avvisi-della-quotazione-warnings) e
[Registrazioni referenziate](api-cotacao-v1.md#registrazioni-referenziate).

### Indirizzo senza validazione nella quotazione
- Nella v2, la quotazione **non** valida CAP, città né stato; solo il paese (e lo `state_code`, quando inviato).
- Nella quotazione v2 il CAP non è richiesto per gli indirizzi in Brasile.
- Lo `state_code` indicato nella quotazione viene riutilizzato nella spedizione v2.

### Nessun endpoint di consultazione
La v2 **non dispone** di `GET /v2/quotation/{quotation}`. Il risultato completo viene fornito nella risposta del `POST`,
quindi **memorizza la risposta** se hai bisogno di mostrarla di nuovo all'utente.

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Regola di business (carico, scatola, articolo, kit) | Leggere `message` e adeguare lo scenario |
| `403` | Senza permesso di quotazione o canale di un altro account | Richiedere il permesso al supporto / rivedere `channel_code` |
| `404` | Articolo/scatola non trovati nel raggruppamento o nessun contratto idoneo | Rivedere registrazioni e scenario |
| `406` | Il carico richiede assistenza personalizzata, o mittente registrato obbligatorio per il tipo di account | Contattare il team commerciale / usare `address_sender.address_code` |
| `422` | Validazione dei campi o HS Code senza tabella di imposta | Correggere i campi elencati in `data` / rivedere `hscode` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- La v2 scambia precisione con semplicità: l'indirizzo completo entra solo nella spedizione.
- I flussi operativi di **status, documenti, annullamento e pianificazione del ritiro** usano gli stessi
  endpoint `/v1/...`, anche per le spedizioni create nella v2 — vedi
  [Tracciamento](api-rastreio.md), [Documenti](api-documentos.md) e [Pianificazione](api-agendamento.md).
- Conserva la risposta del `POST`: nella v2 non esiste un endpoint di riconsultazione.
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
