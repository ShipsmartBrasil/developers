# Articoli

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Gestisce l'anagrafica dei prodotti che spedisci, con peso, dimensioni, HS Code, valori e identificatori (SKU, EAN e NSMP).
- Con l'articolo registrato, la quotazione può referenziarlo solo tramite lo `sku`, senza ripetere tutti i dati.
- È ciò che abilita la [quotazione per articoli](api-cotacao-v2.md#quotazione-di-articoli) (`type: items`, **esclusiva della v2**) e i raggruppamenti automatici delle scatole dell'account.
- Tutte le operazioni di scrittura (`POST`, `PUT`, `DELETE`) sono **asincrone** e vengono convalidate solo durante l'elaborazione.
- L'elenco è **paginato (20 per pagina)** e riflette l'anagrafica in tempo reale (senza cache).

> ⚠️ **Importante:** `POST`, `PUT` e `DELETE` **non convalidano i campi in modo sincrono**.
> L'API risponde `200` confermando la ricezione e convalida ogni articolo durante l'elaborazione.
> Gli articoli non validi vengono **scartati silenziosamente** (senza errori nella risposta HTTP) e il motivo viene registrato nel
> report di importazione degli articoli, nel pannello. Conferma sempre il risultato con `GET /v1/item`.

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

| Header | Obbligatorio | Valori | Predefinito |
|---|---|---|---|
| `Authorization` | **sì** | `Bearer IL_TUO_TOKEN` | - |
| `Content-Type` | sì in `POST`, `PUT` e `DELETE` | `application/json` | - |
| `lang` | no | `pt`, `en`, `es` | `pt` |

Ogni operazione richiede un permesso specifico nel tuo profilo (elencare, registrare/aggiornare e rimuovere articoli).
Senza il permesso, la risposta è `403` con `Has no permission!`.

## Flusso di utilizzo
1. (Facoltativo) Registra le [scatole](api-caixa-v1.md) e annota il `box_code` da collegare agli articoli.
2. Registra gli articoli con `POST /v1/item`.
3. Attendi l'elaborazione asincrona e verifica il report di importazione nel pannello.
4. Conferma con `GET /v1/item` che tutti gli SKU siano stati inseriti.
5. Usa lo `sku` nelle [quotazioni](api-cotacao-v1.md).
6. Aggiorna con `PUT /v1/item` e rimuovi con `DELETE /v1/item`.

## Concetti essenziali

### Perché usare l'anagrafica degli articoli

**Senza anagrafica** — ripetendo tutto in ogni quotazione:
```json
"items": [
  { "box_id": 1, "name": "Maglietta cotone P", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
]
```

**Con anagrafica** — referenziando lo SKU:
```json
"items": [
  { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
]
```

Con `type: items` (**esclusivo della [Quotazione v2](api-cotacao-v2.md#quotazione-di-articoli)**), non devi nemmeno collocare l'articolo in una scatola: la piattaforma compone le scatole in base al
raggruppamento configurato nell'account (vedi [Come usare gli articoli nella quotazione](#come-usare-gli-articoli-nella-quotazione)).

### Identificatori del prodotto

| Campo | Che cos'è | Uso |
|---|---|---|
| `sku` | Il tuo codice interno del prodotto | **Chiave dell'anagrafica**: unico per account, usato per referenziare, aggiornare e rimuovere |
| `ean` | Codice a barre EAN/GTIN | Stampato sull'invoice e inviato come identificatore standard del produttore nelle informazioni regolatorie |
| `nsmp` | *Non Standard Manufacturer Product Id* — identificatore di prodotto **non standardizzato** del produttore (es.: il codice parte/modello del produttore) | Stampato sull'invoice (riga `Mfr:`) e inviato nelle informazioni regolatorie richieste da alcuni corrieri per destinazioni nell'**Unione Europea** |

### Unità di misura

Dimensioni e peso dell'anagrafica sono sempre interpretati in **centimetri** e **chilogrammi**.
La conversione dal sistema `imperial` effettuata nella quotazione vale solo per le misure inviate nel payload della quotazione —
**non** viene applicata ai dati compilati a partire dall'anagrafica. Registra sempre in cm/kg.

### Tipi di dati (l'elaborazione è rigorosa)

La validazione asincrona verifica il **tipo JSON** di ogni campo. I valori con il tipo sbagliato causano lo scarto dell'articolo:

| Campo | Invialo come | Esempio corretto | Esempio scartato |
|---|---|---|---|
| `sku`, `hscode`, `ean`, `nsmp`, `category` | testo (tra virgolette) | `"hscode": "610910"` | `"hscode": 610910` |
| `box_code` | numero intero | `"box_code": 3` | `"box_code": "3"` o `3.0` |
| `height`, `width`, `depth`, `weight`, `unit_value`, `selling_unit_value` | numero | `"weight": 0.25` | `"weight": "0,25"` |
| `is_groupable`, `is_foldable` | booleano JSON | `"is_groupable": false` | `"is_groupable": "false"` (viene salvato, ma trattato come `true` nella quotazione) |

## Endpoint: elencare gli articoli

- **Metodo e rotta:** `GET /v1/item`
- **Quando usarlo:** per consultare l'anagrafica, verificare se l'importazione asincrona è stata completata e controllare i dati di uno SKU.

### Parametri

| Parametro | Posizione | Tipo | Obbligatorio | Predefinito | Descrizione |
|---|---|---|---|---|---|
| `items[skus][]` | query | array di string | no | - | Elenco di SKU (corrispondenza esatta, distingue maiuscole/minuscole). **Vuoto o assente restituisce tutti** |
| `page` | query | integer | no | `1` | Pagina dell'elenco |

- **Elementi per pagina:** 20 (fisso).
- Gli SKU inesistenti vengono ignorati (non generano errori).
- Senza cache: le modifiche elaborate compaiono immediatamente.

### Payload minimo
Senza parametri (restituisce tutti gli articoli, pagina 1):
```
GET /v1/item
```

### Payload massimo
```
GET /v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044&page=1
```

### Esempio di request

```bash
# Tutti gli articoli
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'

# Filtrando per SKU
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

> Il `--globoff` evita che curl interpreti le parentesi quadre della query string.

### Esempio di response

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "sku": "TSH-000-S",
        "name": "Maglietta cotone P",
        "description": "Maglietta 100% cotone manica corta bianca taglia P",
        "category": "610910",
        "country_code": "BR",
        "hscode": "610910",
        "ean": "7894900010015",
        "nsmp": "FAB-TSH-P-001",
        "height": 2,
        "width": 20,
        "depth": 25,
        "weight": 0.25,
        "is_groupable": true,
        "is_foldable": true,
        "box_code": 3,
        "box_identifier": "CX0236",
        "unit_value": 29.9,
        "selling_unit_value": 49.9
      },
      {
        "sku": "SNK-044",
        "name": "Scarpa sportiva 44",
        "description": "Scarpa da corsa uomo numero 44 colore nero",
        "category": "640411",
        "country_code": "BR",
        "hscode": "640411",
        "height": 15,
        "width": 22,
        "depth": 33,
        "weight": 0.8,
        "is_groupable": false,
        "nsmp": null,
        "unit_value": null,
        "selling_unit_value": null
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/item?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 20,
    "prev_page_url": null,
    "to": 2,
    "total": 2
  }
}
```

### Campi della risposta

**Articolo (`data.data[]`):**

| Campo | Tipo | Sempre presente | Descrizione |
|---|---|---|---|
| `sku` | string | sì | Codice univoco del prodotto |
| `name` | string | sì | Nome del prodotto |
| `description` | string | sì | Descrizione del prodotto |
| `category` | string | sì | Categoria (6 caratteri) |
| `country_code` | string | sì | Paese di origine/fabbricazione |
| `hscode` | string | sì | NCM/HS Code |
| `height`, `width`, `depth` | number | sì | Dimensioni (cm) |
| `weight` | number | sì | Peso unitario (kg) |
| `nsmp` | string \| null | **sì** | Identificatore non standardizzato del produttore. Arriva `null` quando non registrato |
| `unit_value` | number \| null | **sì** | Valore unitario predefinito. `null` quando non registrato |
| `selling_unit_value` | number \| null | **sì** | Valore di vendita unitario. `null` quando non registrato |
| `ean` | string | no | Codice EAN/GTIN (compare solo se registrato) |
| `is_groupable` | boolean | no | Consente di raggruppare con altri articoli (compare solo se registrato) |
| `is_foldable` | boolean | no | Articolo pieghevole (compare solo se registrato) |
| `box_code` | integer | no | Scatola predefinita collegata (compare solo se registrata) |
| `box_identifier` | string | no | Scatola predefinita collegata, tramite l'identificatore (compare solo se registrata) |

> I campi vengono restituiti come sono stati salvati (stesso tipo e valore inviati). L'ordine delle chiavi può variare.
> Anche i campi al di fuori dell'elenco documentato inviati nella registrazione vengono salvati e possono comparire nell'elenco:
> **invia solo i campi documentati**.

**Paginazione (`data`):** stessi campi dell'[elenco delle scatole](api-caixa-v1.md#campi-della-risposta)
(`current_page`, `from`, `to`, `per_page` = 20, `total`, `last_page`, URL di navigazione, `path` e `links`).

## Endpoint: creare articoli

- **Metodo e rotta:** `POST /v1/item`
- **Quando usarlo:** per registrare nuovi prodotti, uno o più per richiesta.
- **Elaborazione:** asincrona, in blocco. Ogni articolo viene convalidato singolarmente; quelli validi vengono salvati e quelli non validi
  vengono scartati ed elencati nel report di importazione.
- **Uno SKU già esistente non viene aggiornato:** usa `PUT` per modificarlo.

### Parametri

**Radice del payload:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `items` | array | **sì** | Elenco di articoli. Senza `items`, non viene salvato nulla (la risposta resta `200`) |

**Parametri — `items[]`:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `sku` | string | **sì** | 1 | 50 (consigliato) | - | Codice univoco del prodotto nel tuo account. **Non può ripetersi.** La quotazione accetta SKU fino a 50 caratteri |
| `name` | string | **sì** | 1 | - | - | Nome del prodotto (compare sull'invoice) |
| `description` | string | **sì** | 1 (6 consigliato) | 155 (consigliato) | - | Descrizione del prodotto. **Descrizioni distinte tra gli articoli evitano allerte doganali** |
| `category` | string | **sì** | 6 | 6 | - | Categoria del prodotto. **Esattamente 6 caratteri** (senza accenti: le lettere accentate contano come 2) |
| `country_code` | string | **sì** | 2 | 2 | - | Paese di origine/fabbricazione in ISO 3166-1 alpha-2, maiuscolo (es.: `BR`) |
| `hscode` | string | **sì** | 1 | - | - | NCM/HS Code, **come testo**. Definisce la tabella delle imposte applicata a destinazione |
| `height` | number | **sì** | 0.01 | - | - | Altezza del prodotto (cm) |
| `width` | number | **sì** | 0.01 | - | - | Larghezza del prodotto (cm) |
| `depth` | number | **sì** | 0.01 | - | - | Profondità del prodotto (cm) |
| `weight` | number | **sì** | 0.01 | - | - | Peso unitario (kg) |
| `is_groupable` | boolean | no (**consigliato**) | - | - | `true` | `true` consente di raggruppare l'articolo con altri nella stessa scatola; `false` richiede una scatola esclusiva. Nel raggruppamento `default`, un articolo senza questo campo causa un errore nella quotazione |
| `ean` | string | no | - | 50 (consigliato) | - | Codice EAN/GTIN, **come testo** |
| `nsmp` | string | no | - | 50 (consigliato) | `null` | *Non Standard Manufacturer Product Id*: identificatore non standardizzato del produttore (es.: codice modello/parte). Stampato sull'invoice e usato nelle informazioni regolatorie di alcuni corrieri per destinazioni nell'Unione Europea. La quotazione accetta fino a 50 caratteri |
| `unit_value` | number | no | - | - | - | Valore unitario predefinito. Se assente nell'anagrafica **e** nella quotazione, la quotazione fallisce |
| `selling_unit_value` | number | no | - | - | - | Valore di vendita unitario. Necessario quando la quotazione usa `ior: true` |
| `box_code` | integer | no | - | - | - | Scatola predefinita dell'articolo ([Scatola](api-caixa-v1.md)). Deve essere un **intero** JSON |
| `box_identifier` | string | no | - | 50 | - | Scatola predefinita dell'articolo, tramite l'identificatore testuale |
| `is_foldable` | boolean | no | - | - | `false` | Articolo pieghevole. Nello `smartbox`, gli articoli pieghevoli vengono sistemati per volume; quelli non pieghevoli, per dimensioni |

> **Scatola predefinita (Forma A / Forma B):** collega la scatola tramite `box_code` (Forma A) **o** tramite `box_identifier` (Forma B).
> Non inviare entrambi con valori di scatole diverse. L'esistenza della scatola **non** viene verificata nella registrazione dell'articolo —
> viene verificata solo nella quotazione. Preferisci `box_code`: è il collegamento riconosciuto nella protezione contro la rimozione delle scatole.

> I valori `null` in `unit_value` e `selling_unit_value` vengono ignorati (il campo non viene salvato).

### Payload minimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Maglietta cotone P",
      "description": "Maglietta 100% cotone manica corta bianca taglia P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25
    }
  ]
}
```

> Questo è il payload più piccolo accettato. Per usare l'articolo nella quotazione `type: items` con raggruppamento `default`, includi anche
> `is_groupable` e la scatola predefinita (`box_code` o `box_identifier`).

### Payload massimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Maglietta cotone P",
      "description": "Maglietta 100% cotone manica corta bianca taglia P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "nsmp": "FAB-TSH-P-001",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Scarpa sportiva 44",
      "description": "Scarpa da corsa uomo numero 44 colore nero",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": "FAB-SNK-44-BLK",
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 199.9,
      "selling_unit_value": 349.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

### Esempio di request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Maglietta cotone P", "description": "Maglietta 100% cotone manica corta bianca taglia P", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-001", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true, "box_code": 3}
    ]
  }'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success` (ricevuto) o `error` (errore imprevisto) |
| `message` | string | Messaggio tradotto in base all'header `lang` |

> Il `200` conferma solo la ricezione. Gli articoli scartati compaiono nel report di importazione (vedi
> [Validazione asincrona](#validazione-asincrona--tabella-di-riferimento)).

## Endpoint: aggiornare articoli

- **Metodo e rotta:** `PUT /v1/item`
- **Quando usarlo:** per modificare i dati di prodotti già registrati, in blocco.
- **Chiave di aggiornamento:** lo `sku`. L'articolo registrato con quello SKU riceve i valori inviati.
- **Elaborazione:** asincrona, in blocco, con **la stessa validazione del `POST`**.

> ⚠️ **Invia l'articolo completo.** La validazione del `PUT` è identica a quella del `POST`: `sku`, `name`, `description`, `category`,
> `country_code`, `hscode`, `height`, `width`, `depth` e `weight` sono **obbligatori in ogni aggiornamento**. Un articolo
> inviato solo con lo SKU e il campo modificato viene **scartato** (motivo `name` nel report).

> ⚠️ **Uno SKU inesistente non viene creato.** L'aggiornamento non esegue un "upsert": se lo SKU non esiste, non succede nulla
> (e non viene segnalato nulla). Usa `POST` per creare.

Regole di aggiornamento dei campi facoltativi:

| Situazione nel payload del `PUT` | Risultato nell'anagrafica |
|---|---|
| Campo facoltativo **omesso** | Mantiene il valore attuale |
| Campo facoltativo inviato con un valore | Sostituisce il valore attuale |
| `ean`, `nsmp`, `box_code`, `box_identifier`, `is_foldable` inviati come `null` | Il valore viene cancellato (diventa `null`) |
| `unit_value` o `selling_unit_value` inviati come `null` | **Ignorato**: il valore attuale viene mantenuto (non è possibile cancellare il prezzo tramite il `PUT`; invia un nuovo valore) |

### Parametri

Stessa struttura e stesse regole del [`POST /v1/item`](#endpoint-creare-articoli): radice `items` (array) e i campi di `items[]`.

### Payload minimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Maglietta cotone P",
      "description": "Maglietta premium 100% cotone manica corta bianca P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27
    }
  ]
}
```

### Payload massimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Maglietta cotone P",
      "description": "Maglietta premium 100% cotone manica corta bianca P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "nsmp": "FAB-TSH-P-002",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27,
      "unit_value": 34.9,
      "selling_unit_value": 54.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Scarpa sportiva 44",
      "description": "Scarpa da corsa uomo numero 44 colore nero",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": null,
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 189.9,
      "selling_unit_value": 339.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

> Nel secondo articolo, `"nsmp": null` cancella l'NSMP registrato.

### Esempio di request

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Maglietta cotone P", "description": "Maglietta premium 100% cotone manica corta bianca P", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-002", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success` (ricevuto) o `error` (errore imprevisto) |
| `message` | string | Messaggio tradotto in base all'header `lang` |

## Endpoint: rimuovere articoli

- **Metodo e rotta:** `DELETE /v1/item`
- **Quando usarlo:** per eliminare definitivamente prodotti dall'anagrafica.
- **Elaborazione:** asincrona, in blocco. Gli SKU inesistenti vengono ignorati senza avviso.

### Parametri

| Campo | Posizione | Tipo | Obbligatorio | Min. | Descrizione |
|---|---|---|---|---|---|
| `items` | body | object | **sì** | - | Involucro |
| `items.skus` | body | array di string | **sì** | 1 elemento | Elenco di SKU da rimuovere. Ogni SKU deve essere un testo non vuoto |

> L'assenza di `items.skus` non genera errori nella risposta HTTP (`200`); il problema viene registrato nel report.

### Payload minimo

```json
{
  "items": {
    "skus": ["TSH-000-S"]
  }
}
```

### Payload massimo

```json
{
  "items": {
    "skus": ["TSH-000-S", "TSH-000-M", "SNK-044"]
  }
}
```

### Esempio di request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success` (ricevuto) o `error` (errore imprevisto) |
| `message` | string | Messaggio tradotto in base all'header `lang` |

## Validazione asincrona — tabella di riferimento

Poiché la validazione avviene durante l'elaborazione (`POST` e `PUT`), usa questa tabella per garantire che gli articoli vengano inseriti.
Le regole vengono verificate **in questo ordine** e la **prima** regola violata causa lo scarto dell'articolo.
Nel report di importazione, ogni articolo scartato compare come:

```
sku => TSH-000-S, status => invalid, reason => category
```

| Ordine | Campo | Regola | `reason` nel report |
|---|---|---|---|
| 1 | (articolo) | Deve essere un oggetto | `not_array` |
| 2 | `name` | Obbligatorio, testo non vuoto | `name` |
| 3 | `description` | Obbligatorio, testo non vuoto | `description` |
| 4 | `category` | Obbligatorio, testo, **esattamente 6 caratteri** | `category` |
| 5 | `country_code` | Obbligatorio, testo, **esattamente 2 caratteri** | `country_code` |
| 6 | `sku` | Obbligatorio, testo non vuoto | `sku` |
| 7 | `ean` | Se inviato (non nullo), deve essere testo | `ean` |
| 8 | `nsmp` | Se inviato (non nullo), deve essere testo | `nsmp` |
| 9 | `height` | Obbligatorio, numerico, **≥ 0.01** | `height` |
| 10 | `width` | Obbligatorio, numerico, **≥ 0.01** | `width` |
| 11 | `depth` | Obbligatorio, numerico, **≥ 0.01** | `depth` |
| 12 | `weight` | Obbligatorio, numerico, **≥ 0.01** | `weight` |
| 13 | `hscode` | Obbligatorio, testo non vuoto | `hscode` |
| 14 | `box_code` | Se inviato (non nullo), deve essere un **intero** (non string, non decimale) | `box_code` |
| 15 | `box_identifier` | Se inviato (non nullo), deve essere testo | `box_identifier` |
| 16 | `unit_value` | Se inviato (non nullo), deve essere numerico | `unit_value` |
| 17 | `selling_unit_value` | Se inviato (non nullo), deve essere numerico | `selling_unit_value` |

Regole applicate silenziosamente (senza scarto e senza avviso):

| Situazione | Comportamento |
|---|---|
| `is_groupable` con qualsiasi valore | Accettato e salvato come inviato. Usa un booleano JSON: `"false"` come testo viene trattato come `true` nella quotazione |
| `is_foldable` con qualsiasi valore | Non viene convalidato; salvato come inviato |
| `name`, `description`, `sku`, `ean`, `nsmp` oltre i limiti consigliati | Salvati, ma possono generare errori quando l'articolo viene usato nella quotazione/spedizione |
| `country_code` con 2 caratteri che non è un paese valido | Salvato; l'errore compare solo nella quotazione |
| `box_code`/`box_identifier` di una scatola inesistente | Salvato; l'errore compare solo nella quotazione |
| Campi non documentati | Salvati come inviati |

Altre situazioni registrate nel report:

| Operazione | Situazione | Comportamento |
|---|---|---|
| `POST`/`PUT` | Corpo vuoto (`{}`) | Report: `Itens isn't an array or is empty` |
| `POST`/`PUT` | `items` assente con altri campi nel corpo, oppure `items` non è una lista | L'elaborazione fallisce e viene registrata come errore nel report; non viene salvato nulla |
| `POST`/`PUT` | `items: []` | Non viene salvato nulla e non viene segnalato nulla |
| `POST` | SKU già registrato, o ripetuto nello stesso blocco | L'articolo duplicato viene rifiutato; gli altri articoli validi del blocco vengono salvati e il blocco viene contrassegnato come errore nel report |
| `PUT` | SKU inesistente | Non viene modificato nulla, non viene creato nulla e non viene segnalato nulla |
| `DELETE` | `items.skus` assente o vuoto | Report: `SKUs isn't an array or is empty` |
| `DELETE` | SKU vuoto o non testuale (es.: numero) | Report: `sku => ..., status => invalid, reason => sku_invalid` |
| `DELETE` | SKU inesistente | Ignorato senza avviso |

## Come usare gli articoli nella quotazione

| Quotazione `type` | Payload in `items[]` | Comportamento |
|---|---|---|
| `advanced` (v1 e v2) | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | Sei tu a definire in quale scatola (posizione in `boxes[]`) va l'articolo |
| `items` (**solo v2**) | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | La piattaforma compone le scatole in base al raggruppamento dell'account |
| qualsiasi | `{ "box_id": 1, "name": "Maglietta cotone P", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | Articolo singolo, senza anagrafica |

Quando l'articolo della quotazione ha uno `sku` e manca qualche dato (nome, paese, HS Code, dimensioni, peso, valori o `nsmp`),
i dati mancanti vengono completati dall'anagrafica. I campi inviati nella quotazione hanno la priorità sull'anagrafica.

Uso dell'anagrafica in base al raggruppamento dell'account (quotazione `type: items`, v2):

| Raggruppamento | Requisiti dell'anagrafica | `is_groupable` |
|---|---|---|
| `default` | Ogni articolo deve avere una scatola predefinita (`box_code` o `box_identifier`) e `is_groupable` registrati | `true`: raggruppato con altri articoli della stessa scatola predefinita. `false`: una scatola esclusiva per unità |
| `smartbox` | Le scatole candidate provengono da `boxes[]` nel payload della quotazione | Assente = `true`. `false`: scatola esclusiva. `is_foldable` (assente = `false`) definisce la sistemazione per volume o per dimensioni |
| `all_in_one` | Solo i dati dell'articolo | Assente = `true`. `false`: l'articolo resta fuori dal consolidamento, in una scatola propria |

Vedi la sezione `items[]` in [Quotazione (v1)](api-cotacao-v1.md#parametri--items).

## Regole di business
- Tutte le operazioni di scrittura vengono eseguite in **background**; il `200` conferma solo la ricezione.
- Lo `sku` è **unico** per account: il `POST` non sovrascrive uno SKU esistente.
- Gli articoli sono **per account**: ogni utente vede e modifica solo i propri.
- Il `PUT` richiede l'articolo completo (stessa validazione del `POST`) e non crea nuovi SKU.
- Dimensioni e peso dell'anagrafica sono sempre in **cm** e **kg**.
- `nsmp` è facoltativo; quando compilato, viene stampato sull'invoice e inviato nelle informazioni regolatorie di alcuni
  corrieri per destinazioni nell'Unione Europea. Senza `nsmp`, questo dato resta vuoto.
- L'anagrafica deve essere completa affinché la quotazione funzioni. Messaggi correlati nella quotazione:

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | SKU non registrato e articolo senza i dati minimi nel payload |
| `404` | `O item com o sku :name não existe!` | SKU non registrato nella quotazione `type: items` |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | Articolo senza scatola predefinita valida nel raggruppamento `default` |
| `400` | `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).` | Senza `unit_value` nell'anagrafica e nella quotazione |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | Valore azzerato |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | `hscode` errato per la destinazione |

> I codici HTTP dei messaggi di quotazione sopra riportati seguono la pagina [Quotazione (v1)](api-cotacao-v1.md#errori-comuni).

## Errori comuni

| Status | Significato/Messaggio | Come correggere |
|---|---|---|
| `200` con articoli mancanti | Articolo scartato nella validazione asincrona (`reason => campo` nel report) | Verificare il report nel pannello e la [tabella di validazione](#validazione-asincrona--tabella-di-riferimento) |
| `200` senza effetto nel `PUT` | SKU inesistente o articolo incompleto | Creare con `POST` o inviare l'articolo completo |
| `403` | `Has no permission!` — profilo senza permesso per elencare, registrare/aggiornare o rimuovere articoli | Richiedere il permesso al supporto |
| `422` | `O campo items.skus deve conter um array.` (solo `GET`) | Inviare `items[skus][]` come lista nella query string |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Ripetere a intervalli crescenti; se persiste, contattare il supporto |
| Report | `Itens isn't an array or is empty` | Inviare `items` con almeno un articolo |
| Report | `SKUs isn't an array or is empty` | Nel `DELETE`, inviare `items.skus` con almeno uno SKU |
| Report | `sku => ..., status => invalid, reason => sku_invalid` | Nel `DELETE`, inviare gli SKU come testo non vuoto |

> `POST`, `PUT` e `DELETE` **non restituiscono `422`**: i problemi dei campi compaiono solo nel report di importazione.

## Note finali
- Poiché la validazione non è sincrona, **conferma sempre con `GET /v1/item`** dopo aver registrato, aggiornato o rimosso.
- Invia i tipi JSON corretti: `sku`, `hscode`, `ean` e `nsmp` come testo; `box_code` come intero; booleani senza virgolette.
- Mantieni descrizioni distinte tra prodotti diversi: descrizioni ripetute generano il warning
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` nella quotazione.
- Compila `nsmp` per i prodotti spediti nell'Unione Europea ogni volta che il produttore ha un codice proprio del prodotto.
- Per standardizzare gli imballaggi usati con questi articoli, vedi [Scatola](api-caixa-v1.md).
- Per usare gli articoli nella quotazione, vedi [Quotazione (v1)](api-cotacao-v1.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
