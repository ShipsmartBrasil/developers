# Scatola

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Mantiene la registrazione degli imballaggi che usi con frequenza.
- Una volta registrata, la scatola può essere richiamata nella quotazione tramite `box_code` o `box_identifier`,
  senza ripetere dimensioni e peso in ogni richiesta.
- La registrazione e la rimozione sono **asincrone**: l'API conferma la ricezione ed elabora in background.

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

## Perché usare la registrazione delle scatole

**Senza registrazione** — ripetendo le dimensioni in ogni quotazione:
```json
"boxes": [{ "name": "Scatola S", "height": 20, "width": 20, "depth": 20 }]
```

**Con registrazione** — richiamando la scatola:
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Vantaggi: payload più piccolo, dimensioni standardizzate e uso del raggruppamento `smartbox`
(che sistema automaticamente gli articoli nelle scatole registrate).

## Flusso di utilizzo
1. Registra le scatole con `POST /v1/box`.
2. Attendi qualche istante (elaborazione asincrona).
3. Verifica con `GET /v1/box`.
4. Usa `box_code` o `box_identifier` nelle [quotazioni](api-cotacao-v1.md).
5. Rimuovi le scatole obsolete con `DELETE /v1/box`.

## Endpoint: elencare le scatole

- **Metodo e rotta:** `GET /v1/box`

| Parametro | Posizione | Tipo | Obbligatorio | Predefinito | Descrizione |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array | no | - | Elenco di `box_code` e/o `box_identifier`. **Vuoto o assente restituisce tutte** |
| `page` | query | integer | no | `1` | Pagina dell'elenco |

- **Elementi per pagina:** 15
- L'elenco è **messo in cache per un massimo di 5 minuti**. Una scatola appena registrata può richiedere questo tempo per comparire.

```bash
# Tutte le scatole
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'

# Filtrando per identificatori
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Cartuccia Individuale",
        "width": "65",
        "height": "65",
        "depth": "65",
        "weight": "2.8",
        "created_at": "2026-01-27T10:05:05.000000Z",
        "updated_at": "2026-01-27T10:05:05.000000Z",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `box_code` | integer | **Codice numerico della scatola.** Usalo in `boxes[].box_code` nella quotazione |
| `box_identifier` | string | **Identificatore testuale della scatola.** Usalo in `boxes[].box_identifier` nella quotazione |
| `name` | string | Nome della scatola |
| `height`, `width`, `depth` | number | Dimensioni registrate |
| `weight` | number | Peso dell'imballaggio vuoto |
| `active` | boolean | Scatola attiva. **Le scatole inattive non possono essere usate nella quotazione** |
| `created_at`, `updated_at` | string | Date di creazione e ultima modifica |

## Endpoint: creare o aggiornare le scatole

- **Metodo e rotta:** `POST /v1/box`
- **Comportamento:** se il `box_identifier` (o il `box_code`) esiste già, la scatola viene **aggiornata**.
  In caso contrario, viene **creata**.
- **Elaborazione:** asincrona. La risposta `200` conferma solo la ricezione.

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `boxes` | array | **sì** | 1 elemento | - | Elenco delle scatole da creare/aggiornare |
| `boxes[].box_identifier` | string | **sì** | - | 155 | Identificatore univoco della scatola nella tua base (es.: `CX0236`) |
| `boxes[].name` | string | **sì** | - | 155 | Nome descrittivo |
| `boxes[].height` | number | **sì** | 1 | - | Altezza |
| `boxes[].width` | number | **sì** | 1 | - | Larghezza |
| `boxes[].depth` | number | **sì** | 1 | - | Profondità |
| `boxes[].weight` | number | **sì** | - | - | Peso dell'imballaggio vuoto. Fino a 11 cifre intere e **3 decimali** |
| `boxes[].activated` | boolean | **sì** | - | - | `true` per rendere disponibile la scatola nelle quotazioni |
| `boxes[].box_code` | string | no | - | 50 | Codice della scatola. Indicalo per aggiornare una scatola esistente tramite il codice |

> Le dimensioni seguono il sistema di misura del tuo account/della tua quotazione: **centimetri** con `metric`,
> **pollici** con `imperial`. Il peso è in **chilogrammi** (`metric`) o **libbre** (`imperial`).

### Payload minimo

```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Cartuccia Individuale",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

### Payload massimo

```json
{
  "boxes": [
    {
      "box_code": "1",
      "box_identifier": "CX0236",
      "name": "Cartuccia Individuale",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    },
    {
      "box_identifier": "CX0237",
      "name": "Scatola Master",
      "height": 40,
      "width": 30,
      "depth": 25,
      "weight": 0.85,
      "activated": true
    },
    {
      "box_identifier": "CX0100",
      "name": "Scatola fuori produzione",
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 0.2,
      "activated": false
    }
  ]
}
```

### Esempio di request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "boxes": [
      {"box_identifier": "CX0236", "name": "Cartuccia Individuale", "height": 65, "width": 65, "depth": 65, "weight": 2.8, "activated": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

> **L'elaborazione è asincrona.** Verifica il risultato con `GET /v1/box` (ricordando la cache di 5 minuti)
> oppure segui il report di importazione nel pannello.

## Endpoint: rimuovere le scatole

- **Metodo e rotta:** `DELETE /v1/box`
- **Elaborazione:** asincrona. Al termine, un report via e-mail indica quante scatole sono state rimosse
  e quali sono fallite.

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `boxes` | object | **sì** | Involucro |
| `boxes.box_ids` | array | **sì** | Elenco di `box_code` e/o `box_identifier` da rimuovere |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": ["CX0236", 1]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Regole della rimozione

| Regola | Risultato in caso di fallimento |
|---|---|
| La scatola deve esistere nella tua base | Errore nel report: `box %s not found!` |
| La scatola **non può avere articoli collegati** | Errore nel report: `box %s cannot be deleted, has related item` |

> Per "disattivare" una scatola che ha già articoli collegati, usa `POST /v1/box` con `activated: false`
> invece di rimuoverla.

## Come usare le scatole nella quotazione

| Modo | Payload in `boxes[]` | Quando usarlo |
|---|---|---|
| Per codice | `{ "box_code": 1 }` | Hai conservato il codice numerico |
| Per identificatore | `{ "box_identifier": "CX0236" }` | Usi il tuo stesso codice di imballaggio |
| Manuale | `{ "height": 20, "width": 20, "depth": 20 }` | Imballaggio occasionale, senza registrazione |
| Kit | `{ "kit": "KIT-001" }` | Scatola **e** articoli provengono dalla registrazione del kit |

Vedi la sezione `boxes[]` in [Quotazione (v1)](api-cotacao-v1.md#parametri--boxes).

## Regole di business
- La registrazione e la rimozione vengono elaborate in una **coda asincrona**.
- L'elenco è **messo in cache per 5 minuti** per utente e pagina.
- Le scatole con `activated: false` non compaiono come disponibili nelle quotazioni
  (`A caixa :name não existe ou está desativada!`).
- Le scatole sono **per account**: ogni utente vede solo le proprie.
- Il `box_identifier` deve essere univoco nella tua base — è ciò che permette di aggiornare invece di duplicare.

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Regola di business nell'elaborazione | Leggere `message` e rivedere il payload |
| `403` | Permesso mancante per l'azione | Richiedere il permesso al supporto |
| `422` | Validazione dei campi | Correggere i campi elencati in `data` (dimensioni ≥ 1, `box_identifier` obbligatorio, `activated` booleano) |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- Registrare le scatole è il prerequisito per usare il raggruppamento `smartbox` e il tipo di quotazione `items`.
- Mantieni i `box_identifier` allineati con il tuo ERP/WMS: questo semplifica molto l'integrazione.
- Per registrare i prodotti che vanno dentro le scatole, vedi [Articoli](api-itens-v1.md).
