# Quotazione (v1)

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Calcola nolo, assicurazione e imposte per una spedizione internazionale, su tutti i corrieri abilitati sul tuo account.
- È il **primo passo obbligatorio** del flusso: la spedizione, il tracciamento, i documenti e la pianificazione del ritiro
  sono tutti legati al codice della quotazione (`quotation`).
- La v1 è la versione completa: riceve l'indirizzo (paese, città, CAP/stato) già nella quotazione, valida questo indirizzo
  e dispone di un endpoint per riconsultare il risultato.
- Copre cinque scenari di carico: documento, pacco semplice, pacco avanzato, pallet per LLA e pallet per m³ —
  vedi [Scenari di quotazione](#scenari-di-quotazione). La quotazione **per articoli** (solo `sku`, senza scatole) è esclusiva della
  [Quotazione (v2)](api-cotacao-v2.md#quotazione-di-articoli).

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
2. `POST /v1/quotation` con origine, destinazione, oggetto, tipo e scatole/articoli.
3. La risposta contiene già il risultato calcolato (`data.carriers`) e il codice `data.quotation`.
4. Se devi riconsultarla (o cambiare il livello di dettaglio), usa `GET /v1/quotation/{quotation}/{level}`.
5. Verifica che la quotazione sia [idonea alla spedizione](#quotazioni-non-idonee-alla-spedizione).
6. Scegli un corriere dall'elenco e usa `code` (o `slug`) + `quotation` per creare la
   [spedizione](api-remessa-v1.md) **prima** del `valid_until`.

```
POST /v1/quotation  ──►  quotation + carriers[]
                              │
                              ▼
                   POST /v1/shipping  (carrier_code | slug)
```

## Concetti essenziali

### `object` — cosa viene spedito
| Valore | Significato | Osservazioni |
|---|---|---|
| `doc` | Documento | Quotato per collo, indicando `weight` e `price` di ogni busta/scatola |
| `package` | Pacco/scatola | Flusso più comune |
| `pallet` | Pallet | Accetta dimensioni (LLA) **o** `volume` in m³/ft³ |
| `not_doc` | Legacy | Mantenuto per compatibilità; viene convertito internamente in `package`. **Non usarlo nelle nuove integrazioni** |

### `type` — come viene impostato il calcolo
| Valore | Cosa invii | Quando usarlo |
|---|---|---|
| `simple` | Solo `boxes[]` con `weight` e `price` | Documenti e simulazioni rapide del nolo. **Non è idoneo alla spedizione quando `object` è `package` o `pallet`** |
| `advanced` | `boxes[]` (dimensioni) + `items[]` con `box_id` che indica la posizione della scatola | Flusso standard per la spedizione: controlli tu la distribuzione degli articoli nelle scatole |
| `items` | **Non disponibile nella v1** | Esclusivo della [Quotazione (v2)](api-cotacao-v2.md#quotazione-di-articoli) |

### Matrice di idoneità

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Quotazione e spedizione | Non consigliato | ❌ Solo nella v2 |
| `package` | ⚠️ Solo simulazione (non idonea alla spedizione) | ✅ Quotazione e spedizione | ❌ Solo nella v2 |
| `pallet` | ⚠️ Solo simulazione (non idonea alla spedizione) | ✅ Quotazione e spedizione | ❌ Solo nella v2 |

> **Attenzione:** `type: simple` con `object: package` o `object: pallet` serve solo per la simulazione.
> Quando provi a creare la spedizione, l'API risponde `403` con `Cotação Simples é inelegível para remessa!`.
> Dettagli in [Quotazioni non idonee alla spedizione](#quotazioni-non-idonee-alla-spedizione).

### `tax` — chi paga le imposte
| Valore | Significato |
|---|---|
| `sender` | DDP: il mittente paga imposte e tasse a destinazione |
| `receiver` | DDU: il destinatario paga imposte e tasse a destinazione |

> Se il tuo account ha una **preimpostazione dell'imposta** configurata, essa **sostituisce** il valore inviato in `tax`.
> Vedi [Preimpostazioni applicate nella quotazione](#preimpostazioni-applicate-nella-quotazione).

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

Inviato come **query string** nel `POST` e come **segmento del path** nel `GET`.

## Endpoint: creare la quotazione

- **Metodo e rotta:** `POST /v1/quotation`
- **Query string opzionale:** `level=simple|detailed`
- **Quando usarlo:** ogni volta che hai bisogno di prezzo, tempi e imposte, e prima di creare qualsiasi spedizione.

### Parametri — radice del payload

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Valori accettati | Descrizione |
|---|---|---|---|---|---|---|---|
| `object` | string | **sì** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | Cosa viene spedito |
| `type` | string | **sì** | - | - | - | `simple`, `advanced` | Come viene impostato il calcolo (`items` è esclusivo della v2) |
| `tax` | string | **sì** | - | - | - | `sender`, `receiver` | Chi paga le imposte (può essere sostituito dalla preimpostazione dell'account) |
| `insurance` | boolean | **sì** | - | - | - | `true`, `false` | Attivare l'assicurazione (può essere attivata automaticamente dall'account) |
| `currency_quote` | string | **sì** | 3 | 3 | - | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta in cui viene presentata la quotazione e in cui vengono indicati `unit_value`/`price` |
| `currency_payment` | string | **sì** | 3 | 3 | - | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta in cui verrà effettuato il pagamento |
| `address_sender` | object | **sì** | - | - | - | - | Origine (vedi tabella sotto) |
| `address_receiver` | object | **sì** | - | - | - | - | Destinazione (vedi tabella sotto) |
| `boxes` | array | **sì** | - | - | - | - | Colli della quotazione (vedi tabella sotto) |
| `items` | array | condizionale | - | - | - | - | Obbligatorio quando `type` **non** è `simple` e nessuna scatola usa `kit` |
| `measurement` | string | no | - | - | preset dell'account o `metric` | `metric`, `imperial` | Sistema di misura di dimensioni, pesi e volumi |
| `ior` | boolean | no | - | - | `false` | `true`, `false` | Importer of Record. Quando `true`, viene considerato `items[].selling_unit_value` |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Consegna a un indirizzo residenziale |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Colli non impilabili |
| `description` | string | no | - | 155 | `null` | - | Descrizione libera della quotazione, per tuo riferimento |
| `channel_code` | integer | no | - | - | preset dell'account | - | Codice del canale di vendita registrato e abilitato per il tuo utente |
| `channel_identifier_number` | string | no | - | - | `null` | - | Il tuo identificatore commerciale (n. dell'ordine). **Consigliato**: permette di consultare status/documenti e di annullare per canale |
| `channel_order_number` | string | no | - | 100 | `null` | - | Numero dell'ordine nel canale di vendita |
| `declared_freight` | object | no | - | - | preset dell'account | - | Nolo dichiarato (vedi tabella sotto). Richiede un permesso specifico |
| `manufacturer` | object | no | - | - | `null` | - | Produttore mostrato nella Commercial Invoice (vedi tabella sotto) |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Livello di dettaglio del risultato |

### Parametri — `address_sender` e `address_receiver`

Ci sono **due modi** per indicare l'indirizzo. Scegline uno e non mescolarli:

**Modo A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questo modo | Codice di un indirizzo registrato nel pannello del tuo account. Quando viene usato, non inviare gli altri campi |

**Modo B — indirizzo manuale:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `country_code` | string | **sì** | 2 | 2 | Paese in ISO 3166-1 alpha-2 (es.: `BR`, `US`) |
| `city` | string | **sì** | - | 100 | Città |
| `postal_code` | string | condizionale | - | 12 | CAP/ZIP. Obbligatorio se `state` non viene indicato. **Per gli indirizzi in Brasile è sempre obbligatorio** |
| `state` | string | condizionale | - | - | Nome dello stato/provincia. Obbligatorio se `state_code` e `postal_code` non vengono indicati |
| `state_code` | string | condizionale | - | - | Sigla dello stato/provincia (es.: `SP`, `FL`). Obbligatorio se `state` e `postal_code` non vengono indicati |

> **Regola pratica:** indica sempre `country_code` + `city` + `postal_code` (+ `state_code`, quando lo conosci).
> Questa combinazione è valida per tutti i paesi ed è quella che genera il calcolo più preciso.

> **Mittente registrato obbligatorio per alcuni tipi di account:** gli account di persona fisica e gli account aziendali
> con un unico centro di distribuzione devono indicare il mittente con il **Modo A** (`address_code`).
> Senza di esso, l'API risponde `406` con `Codigo de endereço do remetente não encontrado!`.

### Parametri — `boxes[]`

Ogni elemento dell'array rappresenta un collo (scatola, busta o pallet). Ci sono **tre modi** per descrivere il collo:

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
| `name` | string | no | 4 | 60 | Nome/etichetta del collo (usato nei messaggi di errore) |
| `height` | number | **sì** (eccetto pallet con `volume`) | 1 | - | Altezza |
| `width` | number | **sì** (eccetto pallet con `volume`) | 1 | - | Larghezza |
| `depth` | number | **sì** (eccetto pallet con `volume`) | 1 | - | Profondità (lunghezza) |
| `volume` | number | condizionale | 0.001 | - | **Solo quando `object` è `pallet`.** Alternativa ad altezza/larghezza/profondità, in m³ (`metric`) o ft³ (`imperial`) |

**Campi validi nei modi B e C:**

| Campo | Tipo | Obbligatorio | Min. | Descrizione |
|---|---|---|---|---|
| `weight` | number | **sì** quando `object` è `doc`/`package` **e** `type` è `simple` | > 0 | Peso del collo. In `advanced`, se omesso, è la somma dei pesi degli articoli di quella scatola |
| `price` | number | **sì** quando `object` è `doc`/`package` **e** `type` è `simple` | > 0 | Valore dichiarato del collo. In `advanced`, se omesso, è la somma di `unit_value × quantity` degli articoli di quella scatola |

> `weight` e `price` vengono **ignorati** quando la scatola usa `kit`.

### Parametri — `items[]`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `quantity` | integer | **sì** | 1 | 1000 | Quantità dell'articolo |
| `unit_value` | number | **sì** | > 0 | - | Valore unitario nella valuta della quotazione. Zero o negativo viene rifiutato |
| `box_id` | integer | **sì** | 1 | n. di scatole | **Posizione** della scatola nell'array `boxes` (1 = prima scatola, 2 = seconda...). Non è il `box_code` |
| `sku` | string | no | - | 50 | SKU dell'articolo registrato ([Articoli](api-itens-v1.md)). I campi omessi vengono completati dalla registrazione |
| `ean` | string | no | - | 50 | Codice EAN dell'articolo |
| `nsmp` | string | no | - | 50 | *Non Standard Manufacturer Product Id*: identificatore del prodotto presso il produttore. Stampato sulla invoice e usato nelle informazioni regolatorie richieste da alcuni corrieri per destinazioni nell'Unione Europea |
| `name` | string | **sì** quando `sku` non viene indicato | 6 | 50 | Nome dell'articolo (compare sulla invoice) |
| `description` | string | no | - | 155 | Descrizione dettagliata dell'articolo |
| `country_code` | string | **sì** quando `sku` non viene indicato | 2 | 2 | Paese di origine/fabbricazione dell'articolo |
| `hscode` | string | **sì** quando `sku` non viene indicato | 6 | 10 | NCM/HS Code dell'articolo. Definisce la tabella delle imposte applicata |
| `weight` | number | **sì** quando `sku` non viene indicato | 0.001 | - | Peso unitario dell'articolo |
| `selling_unit_value` | number | no | - | - | Valore di vendita unitario. Considerato quando `ior` è `true`; se omesso, usa il valore della registrazione dell'articolo e, in sua mancanza, `unit_value` |
| `is_groupable` | boolean | no | - | - | Predefinito `true`. Usato nei raggruppamenti automatici delle scatole della quotazione per articoli (v2) |

> **Articoli con lo stesso `name`** generano un `warning` nella risposta
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> La quotazione viene effettuata, ma conviene differenziare i nomi per evitare il blocco in dogana.

### Parametri — `declared_freight`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valore del nolo dichiarato |
| `currency` | string | **sì** quando `value` viene indicato | 3 | 3 | Valuta del nolo dichiarato. Stesso elenco di `currency_quote` |

> Il nolo dichiarato viene applicato solo se il tuo profilo ha il **permesso di nolo dichiarato**. Senza di esso, il campo viene
> ignorato silenziosamente (la quotazione viene effettuata normalmente, senza nolo dichiarato).

### Parametri — `manufacturer`

Dati del produttore stampati sulla Commercial Invoice. Invia `address_code` **o** i campi manuali.

| Campo | Tipo | Consigliato | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `address_code` | integer | alternativa ai campi manuali | - | - | Codice di un indirizzo registrato. Quando viene usato, i dati della registrazione sostituiscono i campi manuali |
| `name` | string | **sì** senza `address_code` | 5 | 30 | Ragione sociale/nome del produttore |
| `country_code` | string | **sì** senza `address_code` | 2 | 2 | Paese |
| `state_code` | string | **sì** senza `address_code` | 1 | 10 | Sigla dello stato |
| `state` | string | no | - | 155 | Nome dello stato |
| `postal_code` | string | **sì** senza `address_code` | - | 12 | CAP/ZIP |
| `city` | string | **sì** senza `address_code` | - | 100 | Città |
| `street` | string | **sì** senza `address_code` | - | 70 | Via |
| `number` | string | no | - | 6 | Numero civico |
| `district` | string | no | - | 30 | Quartiere |
| `complement` | string | no | - | 20 | Complemento |
| `email` | string | **sì** senza `address_code` | - | 60 | E-mail |
| `phone` | string | **sì** senza `address_code` | - | 25 | Telefono |
| `phone_extension` | string | no | - | 6 | Interno |
| `type` | string | **sì** senza `address_code` | - | - | `pf` o `pj` |
| `federal_tax_id` | string | no | - | 20 | CPF/CNPJ o equivalente |
| `state_tax_id` | string | no | - | 20 | Iscrizione statale (Inscrição Estadual) |

> L'API **non rifiuta** un `manufacturer` incompleto nella quotazione, ma i dati vengono stampati sulla invoice:
> invia tutti i campi indicati come consigliati per evitare problemi nello sdoganamento.
> Per gli articoli con `hscode` che inizia con `64` (calzature), il nome del produttore è richiesto dalla dogana.

### Payload minimo

Il payload più piccolo accettato dall'endpoint (quotazione di documento). I payload minimi di ciascun tipo di carico si trovano
in [Scenari di quotazione](#scenari-di-quotazione).

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
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
    "postal_code": "33101"
  },
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
      "nsmp": "MFR-TSH-000-S",
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

> In `type: advanced`, **ogni scatola deve avere almeno un articolo** (eccetto la scatola del `kit`, che porta già i propri).
> Per questo l'esempio associa un articolo alle scatole 1, 2 e 3.

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
        "url_image": "https://files.example.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
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
      }
    ]
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.quotation` | string (UUID) | **Codice della quotazione.** Conservalo: è la chiave di tutto il flusso |
| `data.finished` | boolean | `true` quando la quotazione **è già stata usata** per creare una spedizione; `false` finché è ancora disponibile |
| `data.date` | string | Data/ora della quotazione (`AAAA-MM-GG HH:MM:SS`) |
| `data.description` | string | Descrizione inviata in `description` |
| `data.channel` | string | Nome del canale di vendita, quando è presente `channel_code` |
| `data.channel_identifier_number` | string | Identificatore commerciale inviato nella quotazione |
| `data.object` / `data.type` / `data.tax` / `data.insurance` | - | Valori effettivamente usati nel calcolo (dopo le preimpostazioni dell'account) |
| `data.currency_quote` / `data.currency_payment` | string | Valute della quotazione e del pagamento |
| `data.currency_quote_exchange_value` | string | **Solo in `detailed`.** Cambio applicato alla valuta della quotazione |
| `data.currency_payment_exchange_value` | string | **Solo in `detailed`.** Cambio applicato alla valuta del pagamento |
| `data.warnings` | array | **Opzionale.** Compare solo quando ci sono avvisi (vedi [Avvisi della quotazione](#avvisi-della-quotazione-warnings)) |
| `data.carriers[]` | array | Opzioni di spedizione calcolate |
| `carriers[].code` | integer | Codice del contratto. Usalo in `carrier_code` nella spedizione |
| `carriers[].slug` | string | Identificatore testuale del contratto. Alternativa a `carrier_code` nella spedizione |
| `carriers[].name` | string | Nome commerciale del corriere/contratto |
| `carriers[].url_image` | string | Logo del corriere |
| `carriers[].valid_until` | string | **Validità della quotazione.** La spedizione deve essere creata prima di questa data/ora |
| `carriers[].transit_days` | integer | Tempo di transito stimato in giorni |
| `carriers[].weight_details.type` | string | `real` (peso reale) o `cubed` (peso volumetrico) — quale è stato usato nella tariffazione |
| `carriers[].weight_details.weight` | string | Peso considerato |
| `carriers[].freight` / `freight_final` | string | Nolo prima / dopo le commissioni amministrative |
| `carriers[].insurance` / `insurance_final` | string | Assicurazione prima / dopo le commissioni amministrative |
| `carriers[].tax` / `tax_final` | string | Imposte prima / dopo le commissioni amministrative |
| `carriers[].*_subsidy_rule_code` | integer | **Solo in `detailed`.** Codice della regola di sussidio applicata a nolo, assicurazione o imposta (`null` quando non c'è) |
| `carriers[].*_details` | object | **Solo in `detailed`.** Composizione dettagliata di nolo, assicurazione o imposta (`null` quando non c'è) |
| `carriers[].currency_quote_price` | string | Totale nella valuta della quotazione, prima delle commissioni amministrative |
| `carriers[].currency_quote_amount` | string | Totale da pagare nella valuta della quotazione |
| `carriers[].currency_payment_amount` | string | Totale da pagare nella valuta del pagamento |

## Endpoint: consultare il risultato della quotazione

- **Metodo e rotta:** `GET /v1/quotation/{quotation}/{level}`
- **Quando usarlo:** per riconsultare un risultato già calcolato, cambiare il livello di dettaglio o sapere se la quotazione
  è già stata usata (`finished`).

### Parametri

| Parametro | Posizione | Tipo | Obbligatorio | Valori | Descrizione |
|---|---|---|---|---|---|
| `quotation` | path | string | **sì** | UUID | Codice restituito alla creazione |
| `level` | path | string | no | `simple`, `detailed` | Predefinito `simple` |

### Esempio di request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response

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

> La risposta ha la stessa struttura di `data` del `POST` (eccetto `warnings`).

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Não foi encontrada a cotação especificada!` | Il `quotation` non esiste o non appartiene al tuo account |
| `400` | `Nivel de detalhe da cotação não existe` | `level` diverso da `simple`/`detailed` |
| `403` | `Has no permission!` | Il tuo profilo non ha il permesso di consultazione della quotazione |

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
| [Articoli](#quotazione-di-articoli) | `package` | `items` | - | - | **Esclusivo della v2** |

### Quotazione di Documento

- **Quando usarla:** invio di carte/documenti senza valore commerciale rilevante (contratti, certificati, stampati).
- **Come comporla:** `object: doc` + `type: simple`. Ogni collo (busta/scatola) indica dimensioni, `weight` e `price`.
- **Non usa** `items[]`, `ior` né `manufacturer`: i documenti non passano per il calcolo delle imposte per articolo.
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  [Pacco — avanzato](#quotazione-di-pacco-scatola--avanzata) (oppure, nella v2, [Articoli](api-cotacao-v2.md#quotazione-di-articoli)).

#### Payload minimo

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": true,
  "description": "Ordine con due scatole",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
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
      "selling_unit_value": 349.9
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": false,
  "description": "Due pallet di ricambi industriali",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": false,
  "description": "Pallet per cubatura e pallet per misure",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
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

> **Esclusiva della [Quotazione (v2)](api-cotacao-v2.md#quotazione-di-articoli).** La quotazione per articoli (`type: items`), in cui
> invii solo lo `sku` degli articoli registrati e la piattaforma compone le scatole, non fa parte del contratto della v1.
> Nella v1, indica sempre le scatole e usa [Pacco — avanzato](#quotazione-di-pacco-scatola--avanzata) con `items[].sku`
> per sfruttare la registrazione degli articoli.

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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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

Una quotazione calcolata con successo **non sempre** può diventare una spedizione. Prima di chiamare `POST /v1/shipping`,
verifica le condizioni seguenti. Vengono controllate in quest'ordine al momento della spedizione:

| # | Condizione | Status | Messaggio in caso di errore |
|---|---|---|---|
| 1 | La quotazione esiste ed è stata creata nella **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | La quotazione appartiene al tuo account | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Account con imposta vincolata a DDP: quotazione DDU (`tax: receiver`) senza pagamento approvato | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | La quotazione **non è ancora stata usata** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | Non è `type: simple` con `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | La quotazione **non è scaduta** (`valid_until` del corriere) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | Il corriere scelto è presente in `carriers[]` di questa quotazione | `404` | `Contrato especificado não foi listado para esta cotação!` |

> **Come evitarlo:** usa `advanced` per pacco/pallet, crea la spedizione subito dopo la quotazione,
> usa il `code`/`slug` restituito **in questa** quotazione ed effettua **una quotazione per spedizione**.
> Tutte le validazioni della spedizione si trovano in [Spedizione (v1)](api-remessa-v1.md#quotazioni-non-idonee-alla-spedizione).

## Regole di business

### Preimpostazioni applicate nella quotazione
Le [preimpostazioni dell'account](api-primeiros-passos.md#preimpostazioni-dellaccount-presets) influiscono sul calcolo:

| Preimpostazione | Effetto sulla quotazione |
|---|---|
| Imposta (`tax`) | Quando configurata, **sostituisce** il `tax` inviato. I paesi di destinazione elencati come eccezione ricevono l'opposto del valore predefinito. Se l'account è vincolato a DDP, l'imposta è sempre `sender` |
| Assicurazione automatica | Se attiva, l'assicurazione viene attivata automaticamente quando il valore dichiarato (convertito in USD) raggiunge il minimo configurato, anche con `insurance: false` |
| Nolo dichiarato | Usato quando `declared_freight` non viene inviato. Richiede il permesso di nolo dichiarato |
| Sistema di misura | Usato quando `measurement` non viene inviato. In sua mancanza, `metric` |
| Consegna residenziale / non impilabile | Usati quando `residential_delivery` / `non_stackable` non vengono inviati |
| Canale di vendita | Usato quando `channel_code` non viene inviato |
| Raggruppamento delle scatole | Non si applica alla v1 (usato solo nella quotazione per articoli della v2) |

### Validazione dell'indirizzo
Nella v1 l'indirizzo manuale viene **validato già nella quotazione**:

| Caso | Status | Messaggio |
|---|---|---|
| Indirizzo in Brasile senza `postal_code` | `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` |
| CEP brasiliano inesistente/non valido (quando `state_code` non viene inviato) | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| Indirizzo internazionale non trovato | `404` | `Endereço internacional enviado está errado ou não existe!` |
| Stato/provincia non riconosciuto | `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |
| `address_code` inesistente o di un altro account | `400` | `Nenhum endereco encontrado para o usuário especificado!` |
| Mittente e destinatario identici | `400` | `Endereço de Remetente e Destinatário são iguais!` |

### Restrizioni di carico
La quotazione può essere rifiutata quando il carico non rientra nei contratti del tuo account:

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Scatola senza articolo in `advanced` |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` maggiore del numero di scatole o minore di 1 |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Articolo manuale senza `box_id` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | `weight` superiore al peso massimo della scatola registrata (`simple`) |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Somma degli articoli superiore al peso massimo della scatola registrata |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` inesistente o di un altro account |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` non registrato e dati manuali incompleti |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...` | `unit_value` uguale o minore di zero |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` di un altro account |
| `404` | `Nenhum transportador encontrado para este usuário!` | Account senza contratto abilitato per la rotta/l'oggetto |
| `404` | `Não há contratos que atendem as especificações do pacote enviado...` | Nessun contratto copre la combinazione peso/dimensione/rotta |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Carico fuori dalla fascia automatica — contatta il team commerciale |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | Nessun corriere ha restituito un risultato |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item)...` | HS Code senza tabella di imposta a destinazione — rivedi l'`hscode` |

> I rifiuti relativi al **peso** possono arrivare con `status: "error"` invece di `"warning"`, mantenendo il codice HTTP
> della tabella. Gestiscili in base al codice HTTP e al `message`.

### Avvisi della quotazione (`warnings`)
La quotazione viene effettuata, ma `data.warnings` compare con:

| Avviso | Quando |
|---|---|
| `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` | Due o più articoli con lo stesso `name` |
| `Valores declarados dos itens podem ocasionar problemas aduaneiros` | Valore dichiarato superiore a USD 800 con un corriere espresso (DHL/FedEx) nel risultato |

### Registrazioni referenziate
Quando usi codici di registrazione, questi devono esistere, essere attivi e appartenere al tuo account:

| Campo | Registrazione | Dove gestirla |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Scatola | [Scatola](api-caixa-v1.md) |
| `items[].sku` | Articolo | [Articoli](api-itens-v1.md) |
| `boxes[].kit` | Kit | Pannello di Shipsmart Global |
| `address_*.address_code` / `manufacturer.address_code` | Indirizzo | Pannello di Shipsmart Global |
| `channel_code` | Canale di vendita | Pannello di Shipsmart Global |

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Regola di business (carico, scatola, articolo, kit, indirizzo) | Leggere `message` e adeguare lo scenario |
| `403` | Senza permesso di quotazione o canale di un altro account | Richiedere il permesso al supporto / rivedere `channel_code` |
| `404` | Indirizzo non trovato o nessun contratto idoneo | Rivedere indirizzo e scenario |
| `406` | Il carico richiede assistenza personalizzata, o mittente registrato obbligatorio per il tipo di account | Contattare il team commerciale / usare `address_sender.address_code` |
| `422` | Validazione dei campi o HS Code senza tabella di imposta | Correggere i campi elencati in `data` / rivedere `hscode` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- La risposta del `POST` contiene già il risultato calcolato — non è obbligatorio chiamare il `GET` in seguito.
- `data.warnings` è informativo: la quotazione è stata effettuata, ma c'è un rischio doganale da rivedere.
- Per un payload di quotazione ancora più essenziale (solo paese e stato), vedi [Quotazione V2](api-cotacao-v2.md).
- Per scegliere tra le versioni, vedi [Quotazione & Spedizione: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Per quotazione + spedizione in un'unica chiamata sincrona, vedi [Manifesto](api-manifesto.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
