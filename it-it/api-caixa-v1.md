# Scatola

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Gestisce l'anagrafica degli imballaggi che usi frequentemente (nome, dimensioni e peso dell'imballaggio vuoto).
- Una volta registrata, la scatola può essere referenziata nella quotazione tramite `box_code` o `box_identifier`,
  senza ripetere dimensioni e peso in ogni richiesta.
- La scatola può anche essere collegata a un [articolo registrato](api-itens-v1.md) come scatola predefinita del prodotto.
- La registrazione/aggiornamento e la rimozione sono **asincrone**: l'API convalida il formato, conferma la ricezione ed elabora in background.
- L'elenco è **paginato (15 per pagina)** e mantenuto in cache per un massimo di **5 minuti**.

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
| `Content-Type` | sì in `POST` e `DELETE` | `application/json` | - |
| `lang` | no | `pt`, `en`, `es` | `pt` |

Ogni operazione richiede un permesso specifico nel tuo profilo (elencare, registrare e rimuovere scatole).
Senza il permesso, la risposta è `403` con `Has no permission!`.

## Flusso di utilizzo
1. Registra le scatole con `POST /v1/box`.
2. Attendi qualche istante (elaborazione asincrona). Segui il risultato tramite il report di importazione nel pannello.
3. Conferma con `GET /v1/box` e annota il `box_code` generato (ricorda la cache fino a 5 minuti).
4. Usa `box_code` o `box_identifier` nelle [quotazioni](api-cotacao-v1.md) e nella registrazione degli [articoli](api-itens-v1.md).
5. Rimuovi le scatole obsolete con `DELETE /v1/box` (preferibilmente tramite il `box_code`).

## Concetti essenziali

### Perché usare l'anagrafica delle scatole

**Senza anagrafica** — ripetendo le dimensioni in ogni quotazione:
```json
"boxes": [{ "name": "Scatola P", "height": 20, "width": 20, "depth": 20, "weight": 0.3 }]
```

**Con anagrafica** — referenziando la scatola:
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Vantaggi: payload più piccolo, dimensioni standardizzate e possibilità di collegare ogni articolo registrato a una
scatola predefinita (usata nel raggruppamento `default` della quotazione `type: items`, esclusiva della v2).

### `box_code` x `box_identifier`

| Campo | Chi lo definisce | Tipo nell'elenco | Uso |
|---|---|---|---|
| `box_code` | **La piattaforma** (generato alla registrazione) | integer | Codice numerico della scatola. Modo più affidabile per referenziare la scatola in quotazione, articolo e rimozione |
| `box_identifier` | **Tu** (nel `POST`) | string | Il tuo codice di imballaggio (es.: `CX0236`). Deve essere unico nel tuo account |

> **Evita un `box_identifier` composto solo da numeri** (es.: `"123"`). Nei filtri dell'elenco e nella rimozione,
> qualsiasi valore numerico viene interpretato come `box_code`. Preferisci identificatori con lettere, come `CX0123`.

### Unità di misura

Le dimensioni e il peso delle scatole registrate sono sempre interpretati in **centimetri** e **chilogrammi**.
La conversione dal sistema `imperial` (pollici/libbre) effettuata nella quotazione vale solo per le misure inviate
nel payload stesso della quotazione — **non** viene applicata ai dati provenienti dall'anagrafica. Registra sempre in cm/kg.

## Endpoint: elencare le scatole

- **Metodo e rotta:** `GET /v1/box`
- **Quando usarlo:** per consultare le scatole registrate, scoprire il `box_code` generato e verificare se la registrazione asincrona è stata completata.

### Parametri

| Parametro | Posizione | Tipo | Obbligatorio | Predefinito | Descrizione |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array di string/integer | no | - | Elenco di `box_code` e/o `box_identifier` (si possono mescolare). **Vuoto o assente restituisce tutte** |
| `page` | query | integer | no | `1` | Pagina dell'elenco |

- **Elementi per pagina:** 15 (fisso).
- I valori numerici in `boxes[box_ids][]` vengono cercati come `box_code`; gli altri, come `box_identifier`.
- Gli identificatori inesistenti vengono semplicemente ignorati (non generano errori).

> ⚠️ **Cache di 5 minuti:** il risultato resta in cache fino a 5 minuti per ogni combinazione di account e filtro.
> In questo intervallo:
> - le scatole appena registrate, modificate o rimosse potrebbero non riflettersi nell'elenco;
> - **richieste per pagine diverse con lo stesso filtro possono restituire la stessa pagina già consultata**
>   (il numero di pagina non differenzia la cache). Se hai più di 15 scatole, preferisci consultare tramite
>   `boxes[box_ids][]` oppure attendi la scadenza della cache tra una pagina e l'altra.

### Payload minimo
Senza parametri (restituisce tutte le scatole, pagina 1):
```
GET /v1/box
```

### Payload massimo
```
GET /v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1&page=1
```

### Esempio di request

```bash
# Tutte le scatole
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'

# Filtrando per identificatore e per codice
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
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
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Astuccio Singolo",
        "height": 65,
        "width": 65,
        "depth": 65,
        "weight": 2.8,
        "created_at": "2026-01-27 10:05:05",
        "updated_at": "2026-01-27 10:05:05",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/box?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campi della risposta

**Scatola (`data.data[]`):**

| Campo | Tipo | Descrizione |
|---|---|---|
| `box_code` | integer | **Codice numerico della scatola**, generato dalla piattaforma. Usalo in `boxes[].box_code` nella quotazione e in `box_code` dell'articolo |
| `box_identifier` | string \| null | **Il tuo identificatore testuale della scatola**. Usalo in `boxes[].box_identifier` nella quotazione. Può arrivare `null` per le scatole create senza identificatore dal pannello |
| `name` | string | Nome della scatola |
| `height`, `width`, `depth` | number | Dimensioni registrate (cm) |
| `weight` | number | Peso dell'imballaggio vuoto (kg) |
| `created_at`, `updated_at` | string | Date di creazione e ultima modifica, nel formato `AAAA-MM-GG HH:MM:SS` |
| `active` | boolean | Indica se la scatola è attiva. Tratta anche `1`/`0` come vero/falso |

**Paginazione (`data`):**

| Campo | Tipo | Descrizione |
|---|---|---|
| `current_page` | integer | Pagina attuale |
| `from`, `to` | integer \| null | Posizione del primo e dell'ultimo record della pagina |
| `per_page` | integer | Record per pagina (15) |
| `total` | integer | Totale dei record trovati |
| `last_page` | integer | Numero dell'ultima pagina |
| `first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url` | string \| null | URL di navigazione (mantengono i filtri della query) |
| `path` | string | URL di base dell'elenco |
| `links` | array | Link di navigazione pronti (`url`, `label`, `active`) |

## Endpoint: creare o aggiornare scatole

- **Metodo e rotta:** `POST /v1/box`
- **Quando usarlo:** per registrare nuove scatole o modificare scatole esistenti, in blocco.
- **Comportamento:** per **aggiornare** una scatola esistente, indica il suo `box_identifier` (o il `box_code`).
  Se la scatola non esiste, viene **creata**.
- **Elaborazione:** l'API convalida il formato di tutti i campi (errore `422` se qualcosa non va) e, se è tutto corretto,
  risponde `200` confermando **solo la ricezione**. La registrazione avviene in background e il risultato si trova nel
  report di importazione delle scatole, nel pannello.

### Parametri

**Radice del payload:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `boxes` | array | **sì** | 1 elemento | - | - | Elenco di scatole da creare/aggiornare. Un array vuoto viene rifiutato |

**Parametri — `boxes[]`:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `box_identifier` | string | **sì** | - | 155 (usa fino a **50**) | - | Il tuo identificatore univoco della scatola (es.: `CX0236`). La validazione accetta fino a 155 caratteri, ma gli identificatori con più di 50 caratteri **non vengono salvati correttamente** |
| `name` | string | **sì** | - | 155 | - | Nome descrittivo della scatola |
| `height` | number | **sì** | 1 | - | - | Altezza in cm |
| `width` | number | **sì** | 1 | - | - | Larghezza in cm |
| `depth` | number | **sì** | 1 | - | - | Profondità in cm |
| `weight` | number | **sì** | - | 11 cifre intere + 3 decimali | - | Peso dell'imballaggio vuoto in kg. Formato accettato: fino a 11 cifre intere e fino a **3 cifre decimali**, separatore `.`, senza segno (es.: `0.85`, `2.8`, `12`). Indica un valore maggiore di zero |
| `activated` | boolean | **sì** | - | - | - | `true` per scatola attiva, `false` per inattiva. Accetta anche `1`/`0` |
| `box_code` | **string** | no | - | 50 | - | Codice della scatola già esistente, per aggiornarla tramite il codice. **Invialo come testo** (es.: `"1"`): se inviato come numero, viene rifiutato con `422` |

> **Attenzione al tipo di `box_code` nel `POST`:** qui è **string** (`"1"`). Nella quotazione, nella registrazione degli articoli e nella
> rimozione, è **integer** (`1`).

### Payload minimo

```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Astuccio Singolo",
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
      "name": "Astuccio Singolo",
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
      {"box_identifier": "CX0236", "name": "Astuccio Singolo", "height": 65, "width": 65, "depth": 65, "weight": 2.8, "activated": true}
    ]
  }'
```

### Esempio di response

**Successo (`200`):**
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

**Errore di validazione (`422`):**
```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "boxes.0.height": ["O campo boxes.0.height deve conter um número superior ou igual a 1."],
    "boxes.0.box_code": ["O campo boxes.0.box code deve ser uma string."]
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success`, `warning` (validazione/regola) o `error` (imprevisto) |
| `message` | string | Messaggio tradotto in base all'header `lang` |
| `data` | object | Solo negli errori di validazione: mappa `campo` → elenco di messaggi. Il campo arriva con l'indice della scatola (`boxes.0.*` = prima scatola) |

> **Il `200` non garantisce che la scatola sia stata salvata.** Conferma il risultato con `GET /v1/box` (ricordando la cache
> fino a 5 minuti) o tramite il report di importazione nel pannello.

## Endpoint: rimuovere scatole

- **Metodo e rotta:** `DELETE /v1/box`
- **Quando usarlo:** per eliminare definitivamente le scatole che non vengono più usate.
- **Elaborazione:** l'API convalida il formato e risponde `200`; la rimozione viene eseguita in background.
  Al termine, ricevi **un report via e-mail** (all'e-mail dell'account) con il totale delle scatole elaborate,
  il totale rimosso e l'elenco degli errori.
- La rimozione è **definitiva**. Per disattivare soltanto, usa `POST /v1/box` con `activated: false`.

### Parametri

| Campo | Posizione | Tipo | Obbligatorio | Min. | Descrizione |
|---|---|---|---|---|---|
| `boxes` | body | object | **sì** | - | Involucro |
| `boxes.box_ids` | body | array di integer/string | **sì** | 1 elemento | Elenco di scatole da rimuovere. **Usa il `box_code` come numero intero** (es.: `1`) |

> ⚠️ **Rimuovi tramite il `box_code` (intero).** Sebbene la documentazione interattiva accetti `box_identifier` in questo elenco,
> la rimozione tramite identificatore testuale **non è affidabile** al momento:
> - la scatola può essere segnalata come `box CX0236 not found!` anche se esiste;
> - il controllo degli articoli collegati riconosce solo il collegamento fatto tramite `box_code` — una scatola collegata ad articoli solo
>   tramite `box_identifier` può essere rimossa.
>
> Invia inoltre il `box_code` come **numero** (`1`) e non come testo (`"1"`): con il testo, il controllo degli articoli
> collegati potrebbe non individuare il collegamento. Consulta il `box_code` con `GET /v1/box` prima di rimuovere.

### Payload minimo

```json
{
  "boxes": {
    "box_ids": [1]
  }
}
```

### Payload massimo

```json
{
  "boxes": {
    "box_ids": [1, 2, 3, 15]
  }
}
```

### Esempio di request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": [1, 2]}}'
```

### Esempio di response

**Ricevuto (`200`):**
```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> Nonostante il testo, il messaggio sopra conferma solo la **ricezione**. Il risultato reale arriva nel report via e-mail.

**Errore imprevisto (`500`):**
```json
{
  "status": "error",
  "message": "Algo inesperado aconteceu. Tente novamente mais tarde!"
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success`, `warning` o `error` |
| `message` | string | Messaggio tradotto in base all'header `lang` |
| `data` | object | Solo negli errori di validazione: mappa `campo` → elenco di messaggi |

### Regole della rimozione (report via e-mail)

Ogni scatola di `box_ids` viene valutata singolarmente; un errore non impedisce la rimozione delle altre.

| Regola | Messaggio nel report in caso di errore |
|---|---|
| La scatola deve esistere nel tuo account | `Box Error: box %s not found!` (`%s` = valore inviato) |
| La scatola **non può avere articoli collegati** (`box_code` dell'articolo) | `Box Error: box %s cannot be deleted, has related item` |

> Per dismettere una scatola che ha già articoli collegati, modifica la scatola predefinita di tali articoli
> ([`PUT /v1/item`](api-itens-v1.md#endpoint-aggiornare-articoli)) prima di rimuoverla, oppure disattiva semplicemente la scatola
> con `POST /v1/box` e `activated: false`.

## Come usare le scatole nella quotazione

| Forma | Payload in `boxes[]` | Quando usarla |
|---|---|---|
| Per codice | `{ "box_code": 1 }` | Hai conservato il codice numerico (integer) |
| Per identificatore | `{ "box_identifier": "CX0236" }` | Usi il tuo codice di imballaggio |
| Manuale | `{ "height": 20, "width": 20, "depth": 20, "weight": 0.3 }` | Imballaggio singolo, senza anagrafica |
| Kit | `{ "kit": "KIT-001" }` | Scatola **e** articoli provengono dall'anagrafica del kit |

Usa **una sola** forma per scatola (non mescolare `box_code` con `box_identifier` o con dimensioni manuali nella stessa posizione).
Una scatola referenziata che non esiste genera un errore nella quotazione:

| Situazione | Status | Messaggio |
|---|---|---|
| `box_code`/`box_identifier` inesistente sulla piattaforma | `422` | `O valor selecionado para o campo boxes.0.box code é inválido.` (o `box identifier`) |
| La scatola non appartiene al tuo account | `400` | `A caixa :name não foi encontrada para o usuário especificado!` |

Uso delle scatole registrate in base al raggruppamento configurato nell'account (quotazione `type: items`, esclusiva della v2):

| Raggruppamento | Come vengono usate le scatole registrate |
|---|---|
| `default` | Ogni articolo registrato deve avere una scatola predefinita (`box_code` o `box_identifier` nella [registrazione dell'articolo](api-itens-v1.md)). Senza scatola collegata: `A caixa com o box_code/box_identifier :name não existe ou está desativada` (HTTP `404`) |
| `smartbox` | Usa le scatole indicate con dimensioni in `boxes[]` nel payload della quotazione. Senza scatola valida: `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (HTTP `400`) |
| `all_in_one` | Consolida gli articoli; non dipende dalla scatola predefinita dell'articolo |

Vedi la sezione `boxes[]` in [Quotazione (v1)](api-cotacao-v1.md#parametri--boxes).

## Regole di business
- Creazione, aggiornamento e rimozione vengono elaborati in **background**; il `200` conferma solo la ricezione.
- L'elenco resta in **cache fino a 5 minuti** per account e filtro (vedi l'avviso sulla paginazione sopra).
- Le scatole sono **per account**: ogni utente elenca, usa e rimuove solo le proprie.
- `box_identifier` deve essere unico nel tuo account — è ciò che consente di aggiornare invece di duplicare. Usa al massimo 50 caratteri ed evita valori solo numerici.
- Dimensioni e peso registrati sono sempre in **cm** e **kg**, indipendentemente dal `measurement` della quotazione.
- L'indicatore `active` è informativo nell'elenco: **non referenziare scatole inattive** nelle quotazioni. Al momento, la quotazione
  non blocca in modo garantito una scatola inattiva referenziata tramite `box_code`/`box_identifier`.
- Una scatola con articoli collegati (tramite il `box_code` dell'articolo) non viene rimossa.
- La rimozione è definitiva e viene notificata via e-mail.

## Errori comuni

| Status | Significato/Messaggio | Come correggere |
|---|---|---|
| `403` | `Has no permission!` — il profilo non ha il permesso di elencare, registrare o rimuovere scatole | Richiedere il permesso al supporto |
| `422` | `O campo boxes é obrigatório.` — `boxes` assente o vuoto (`POST`/`DELETE`) | Inviare almeno una scatola (`POST`) o l'involucro `boxes` (`DELETE`) |
| `422` | `O campo boxes deve conter um array.` — `boxes` non è una lista/oggetto | Correggere la struttura del payload |
| `422` | `O campo boxes.0.box identifier é obrigatório.` / `O campo boxes.0.name é obrigatório.` | Indicare `box_identifier` e `name` in tutte le scatole |
| `422` | `O campo boxes.0.box identifier não pode conter mais de 155 caracteres.` / `O campo boxes.0.name não pode conter mais de 155 caracteres.` | Accorciare il valore (per `box_identifier`, usa fino a 50) |
| `422` | `O campo boxes.0.height deve conter um número superior ou igual a 1.` (idem `width`/`depth`) | Indicare dimensioni ≥ 1 cm |
| `422` | `O campo boxes.0.height deve conter um valor numérico.` (idem `width`/`depth`/`weight`) | Inviare numeri, senza unità nel valore |
| `422` | `O formato do valor informado no campo boxes.0.weight é inválido.` | Usare fino a 3 cifre decimali, separatore `.` e senza segno |
| `422` | `O campo boxes.0.activated deve conter o valor verdadeiro ou falso.` | Inviare `true`/`false` |
| `422` | `O campo boxes.0.box code deve ser uma string.` | Nel `POST`, inviare `box_code` come testo (`"1"`) |
| `422` | `O campo boxes.0.box code não pode conter mais de 50 caracteres.` | Verificare il `box_code` |
| `422` | `O campo boxes.box ids é obrigatório.` / `O campo boxes.box ids deve conter um array.` (`DELETE`; nel `GET`, solo il secondo) | Inviare `box_ids` come lista non vuota |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Ripetere a intervalli crescenti; se persiste, contattare il supporto |
| Report (e-mail) | `Box Error: box %s not found!` | Verificare il `box_code` con `GET /v1/box` |
| Report (e-mail) | `Box Error: box %s cannot be deleted, has related item` | Scollegare la scatola dagli articoli o semplicemente disattivarla |

> Nei messaggi di validazione, l'indice (`boxes.0`, `boxes.1`...) indica la posizione della scatola nell'array, a partire da `0`,
> e il `_` del nome del campo compare come spazio (`box identifier`).

## Note finali
- Registrare le scatole e collegarle agli articoli è il prerequisito per la [quotazione per articoli della v2](api-cotacao-v2.md#quotazione-di-articoli) (`type: items`) con raggruppamento `default`.
- Mantieni i `box_identifier` allineati con il tuo ERP/WMS: questo semplifica molto l'integrazione.
- Conserva il `box_code` restituito nell'elenco: è il riferimento più affidabile per quotazione, articoli e rimozione.
- Per registrare i prodotti che vanno dentro le scatole, vedi [Articoli](api-itens-v1.md).
- Per referenziare le scatole nella quotazione, vedi [Quotazione (v1)](api-cotacao-v1.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
