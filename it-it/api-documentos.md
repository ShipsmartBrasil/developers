# Documenti della Spedizione

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Recupera i documenti generati dalla piattaforma e dal corriere dopo l'elaborazione della spedizione:
  AWB (etichetta), invoice, packing list, dichiarazione e altri.
- Restituisce anche gli **allegati** che hai inviato alla creazione/aggiornamento della spedizione.
- Ogni documento arriva con una `url` per il download e, facoltativamente, con il contenuto in `base64`.
- Gli account con un permesso specifico ricevono anche i **documenti incorporati** (tutti i PDF in un unico file e tutte le etichette ZPL in un unico contenuto).
- Funziona per le spedizioni create in **v1 e in v2** (le rotte dei documenti esistono solo nella v1).

## URL di base
- Produzione: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Documentazione interattiva: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticazione
```
Authorization: Bearer IL_TUO_TOKEN
lang: pt
```

| Header | Obbligatorio | Valori | Predefinito | Osservazione |
|---|---|---|---|---|
| `Authorization` | **sì** | `Bearer <token>` | - | Token del tuo account. Il profilo deve avere il permesso di consultazione dei documenti della spedizione |
| `lang` | no | `pt`, `en`, `es` | `pt` | Lingua dei messaggi |

## Flusso di utilizzo
1. Crea la spedizione ([Spedizione v1](api-remessa-v1.md) o [Spedizione v2](api-remessa-v2.md)).
2. Consulta lo [status](api-rastreio.md) fino a quando la spedizione compare nel gruppo `shipped` — è in quel momento che
   i documenti iniziano a esistere.
3. Consulta i documenti tramite `{shipping}` o tramite il canale.
4. Scarica il file dalla `url` **immediatamente** o decodifica il `base64`, e conservalo dalla tua parte.

> **Prima di `shipped` la consultazione può restituire `400 Nenhum documento encontrado com o código especificado!`.**
> È un comportamento previsto: attendi l'elaborazione presso il corriere. Se la spedizione ha allegati, la risposta può essere
> `200` con i soli allegati (vedi [Regole di business](#regole-di-business)).

## Concetti essenziali

### Tipi di documento filtrabili (`documents`)

| Valore in `documents` | Chiave in `data.docs` | Contenuto |
|---|---|---|
| `awb_pdf` | `AWB` | Etichetta/AWB in PDF |
| `awb_zpl` | `AWB_ZPL` e `AWB_ZPL_UPS` | Etichetta/AWB in ZPL (stampante termica). Per alcuni corrieri la chiave è `AWB_ZPL_UPS` |
| `awb_prc_pdf` | `AWBPRC` | Etichetta AWB PRC in PDF, emessa per alcuni corrieri |
| `awb_prc_zpl` | `AWBPRCZPL` | Etichetta AWB PRC in ZPL |
| `awb_prc_fedex_pdf` | `AWBPRCFedex` | Etichetta AWB PRC FedEx in PDF |
| `awb_prc_fedex_zpl` | `AWBPRCFedexZPL` | Etichetta AWB PRC FedEx in ZPL |
| `invoice_pdf` | `Invoice` | Commercial Invoice in PDF |
| `packing_list_pdf` | `PackingList` | Packing list in PDF |
| `declaration_pdf` | `Declaration` | Dichiarazione di contenuto in PDF |
| `non_dangerous_pdf` | `NonDangerousObject` | Dichiarazione di merce non pericolosa in PDF |

> **Le chiavi di `data.docs` non contengono spazi** (`PackingList`, `NonDangerousObject`, `AWBPRC`...), sebbene la
> documentazione interattiva le mostri con gli spazi (`Packing List`, `Non-Dangerous Object`, `AWB PRC`...).
> Programma la lettura in base alle chiavi di questa tabella.

Esempio di filtro:
```
?documents=invoice_pdf,awb_pdf,awb_zpl
```

> Non tutti i documenti esistono in ogni spedizione: l'insieme dipende dal corriere, dal paese di destinazione e dal
> tipo di oggetto. Compaiono in `data.docs` solo i documenti effettivamente generati, e ogni chiave compare
> **una sola volta**.

### Documenti incorporati (`embedded_docs`)

| `type` | Contenuto | Condizione |
|---|---|---|
| `pdf` | Tutti i documenti PDF restituiti (dopo il filtro `documents`) uniti in un unico PDF. Il numero di copie di ciascun documento e l'inclusione degli allegati PDF seguono la configurazione di incorporazione del tuo account | La spedizione deve essere già stata elaborata presso il corriere |
| `zpl` | Tutte le etichette ZPL restituite (dopo il filtro `documents`) concatenate, separate da un'interruzione di riga | Deve esistere almeno un documento ZPL |

I documenti incorporati vengono generati solo quando **tutte** le condizioni seguenti sono vere:
`embedded=true`, `base64=true`, il tuo account ha il permesso dei documenti incorporati e la spedizione ha
almeno un documento restituito. In caso contrario, `embedded_docs` arriva come lista vuota (`[]`).

### Stampa delle etichette

| Formato | Quando usarlo |
|---|---|
| `awb_pdf` | Stampante comune (A4/Letter). Apri la `url` o decodifica il `base64` |
| `awb_zpl` | Stampante termica Zebra. Invia il contenuto **puro** (dopo aver decodificato il Base64) direttamente alla coda della stampante |

> Il contenuto ZPL è testo. Non convertirlo né rielaborarlo: invialo esattamente come è arrivato dopo la decodifica.

### Gestione degli allegati

| Operazione | Rotta | Pagina |
|---|---|---|
| Inviare allegati alla creazione | `POST /v1/shipping` (campo `attachments`) | [Spedizione (v1)](api-remessa-v1.md) |
| Inviare allegati in seguito | `PUT /v1/shipping/{shipping}` | [Spedizione (v1)](api-remessa-v1.md#endpoint-aggiornare-la-spedizione) |
| Elencare gli allegati | `GET /v1/shipping/{shipping}/docs?attachments=true&base64=true` | Questa pagina |
| Rimuovere allegati | `DELETE /v1/shipping/{shipping}/docs` | [Spedizione (v1)](api-remessa-v1.md#endpoint-rimuovere-allegati) |

Limiti degli allegati:

| Limite | Valore |
|---|---|
| Formati accettati | `PDF`, `XML`, `JPEG`, `PNG` |
| Dimensione per file | **5 MB** |
| Dimensione totale della richiesta | **90 MB** |

> Gli allegati **vengono restituiti solo con `base64=true`**. Con `base64=false`, `data.attachments` arriva sempre vuoto.

## Endpoint: consultare i documenti per spedizione

- **Metodo e rotta:** `GET /v1/shipping/{shipping}/docs`
- **Quando usarlo:** è il modo più diretto. Accetta l'UUID della quotazione, l'AWB o la chiave della NF-e.

### Parametri

| Nome | Posizione | Tipo | Obbligatorio | Dimensione | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | - | Identificatore della spedizione. Vedi le forme accettate qui sotto |
| `attachments` | query | boolean | no | - | `true` | Includere gli allegati inviati da te. Ha effetto solo con `base64=true` |
| `embedded` | query | boolean | no | - | `true` | Includere i documenti incorporati. Ha effetto solo con `base64=true` e con il permesso specifico |
| `base64` | query | boolean | no | - | `true` | Includere il contenuto dei file in Base64, oltre alla `url`. Con `false`, allegati e documenti incorporati non vengono restituiti |
| `documents` | query | string | no | - | tutti | Elenco di tipi separati da virgole (valori della tabella [Tipi di documento filtrabili](#tipi-di-documento-filtrabili-documents)). Vuoto o assente: tutti |

Modi di indicare `{shipping}` (usane solo uno):

| Forma | Valore | Come viene riconosciuto | Osservazioni |
|---|---|---|---|
| A | UUID della quotazione | Contiene un trattino (`-`) | Forma consigliata |
| B | Chiave della NF-e | Esattamente 44 cifre numeriche | Considera la spedizione più recente con questa chiave, escludendo le spedizioni annullate |
| C | AWB (numero di tracciamento) | Qualsiasi altro valore | Solo spedizioni create negli **ultimi 90 giorni**; considera la più recente con questo AWB |

> Poiché qualsiasi valore con un trattino viene trattato come UUID, un AWB che contiene `-` non viene trovato da questa rotta.
> In tal caso, usa l'UUID della quotazione.

Regole dei parametri di query:

| Regola | Comportamento |
|---|---|
| Valori booleani | Sono considerati veri: `true`, `1`, `on`, `yes`. **Qualsiasi altro valore, compreso il vuoto** (`attachments=`), è considerato `false` |
| `documents` con valore sconosciuto | Il valore viene ignorato. Se **nessun** valore è valido, il filtro non viene applicato e vengono restituiti tutti i documenti |
| `documents` con lettere maiuscole | Non viene riconosciuto (i valori distinguono tra maiuscole e minuscole) |
| `documents` ripetuto (`documents=a&documents=b`) | Viene considerato solo l'**ultimo**. Inviali tutti in un unico parametro, separati da virgole |
| `documents[]=...` | Non supportato: la richiesta fallisce con `500` |

> **Suggerimento sulle prestazioni:** invia `base64=false` quando scarichi dalla `url`. Il payload diventa molto più piccolo
> e la risposta molto più rapida. Ricorda che, in questo modo, allegati e documenti incorporati non arrivano nella risposta.

### Payload minimo

Endpoint senza corpo. È sufficiente il parametro di path (tutti i parametri di query assumono il valore predefinito):

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs
```

### Payload massimo

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=awb_pdf,awb_zpl,awb_prc_pdf,awb_prc_zpl,awb_prc_fedex_pdf,awb_prc_fedex_zpl,invoice_pdf,packing_list_pdf,declaration_pdf,non_dangerous_pdf
```

### Esempio di request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf,awb_zpl' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

Solo gli URL, senza contenuto Base64:

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response

`200 OK` — con `base64=true`, allegati e permesso dei documenti incorporati:

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": {
      "Invoice": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      },
      "PackingList": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_packing_list.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Declaration": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_declaration.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "NonDangerousObject": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_non_dangerous_object.pdf",
        "base64": "JVBERi0xLjcK..."
      }
    },
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." },
      { "id": 2, "type": "xml", "description": "XML della Nota Fiscal", "base64": "PD94bWwg..." }
    ],
    "embedded_docs": [
      { "type": "pdf", "base64": "JVBERi0xLjcK..." },
      { "type": "zpl", "base64": "XlhBDQpeTUNZ..." }
    ],
    "carrier": "Contratto Esempio Espresso",
    "tracking_id": "1234567890123"
  }
}
```

`200 OK` — con `base64=false`:

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": {
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": null
      }
    },
    "attachments": [],
    "embedded_docs": [],
    "carrier": "Contratto Esempio Espresso",
    "tracking_id": "1234567890123"
  }
}
```

`200 OK` — spedizione ancora senza documenti, ma con allegati (`base64=true`):

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": [],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." }
    ],
    "embedded_docs": []
  }
}
```

`400 Bad Request` — nessun documento né allegato:

```json
{
  "status": "warning",
  "message": "Nenhum documento encontrado com o código especificado!"
}
```

### Campi della risposta

| Campo | Tipo | Presenza | Descrizione |
|---|---|---|---|
| `status` | string | sempre | `success`, `warning` o `error` |
| `message` | string | sempre | Messaggio tradotto in base all'header `lang` |
| `data.docs` | object \| array | sempre | Mappa `chiave del documento → { url, base64 }`. Le chiavi sono quelle della tabella [Tipi di documento filtrabili](#tipi-di-documento-filtrabili-documents). **Quando non ci sono documenti, arriva come lista vuota (`[]`)** |
| `data.docs.*.url` | string | sempre | URL per il download del file. Può essere temporaneo (vedi [Regole di business](#regole-di-business)) |
| `data.docs.*.base64` | string \| null | sempre | Contenuto in Base64. `null` quando `base64=false` |
| `data.attachments` | array | sempre | Allegati inviati da te. Lista vuota quando `attachments=false`, `base64=false` o in assenza di allegati |
| `data.attachments[].id` | integer | - | **ID dell'allegato.** Usalo per rimuoverlo con `DELETE /v1/shipping/{shipping}/docs` |
| `data.attachments[].type` | string | - | Estensione del file: `pdf`, `xml`, `jpg` o `png` |
| `data.attachments[].description` | string | - | Descrizione indicata nell'invio dell'allegato |
| `data.attachments[].base64` | string | - | Contenuto in Base64 |
| `data.embedded_docs` | array | sempre | Documenti incorporati. Lista vuota quando non si applicano (vedi [Documenti incorporati](#documenti-incorporati-embedded_docs)) |
| `data.embedded_docs[].type` | string | - | `pdf` o `zpl` |
| `data.embedded_docs[].base64` | string | - | Contenuto in Base64 |
| `data.carrier` | string | quando ci sono documenti | Nome del contratto che ha elaborato la spedizione oppure, in assenza di contratto, il nome del corriere |
| `data.tracking_id` | string \| null | quando ci sono documenti | Numero di tracciamento/AWB |

## Endpoint: consultare i documenti per canale

- **Metodo e rotta:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`
- **Quando usarlo:** quando hai conservato solo il numero del tuo ordine (indicato nella quotazione).

### Parametri

| Nome | Posizione | Tipo | Obbligatorio | Dimensione | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `channel_identifier_number` | path | string | **sì** | - | - | Identificatore dell'ordine nel canale, esattamente come indicato nella quotazione |
| `channel_code` | path | integer | no | - | assente | Codice del canale di vendita indicato nella quotazione. Deve essere un numero intero |
| `attachments` | query | boolean | no | - | `true` | Come in [consultare i documenti per spedizione](#endpoint-consultare-i-documenti-per-spedizione) |
| `embedded` | query | boolean | no | - | `true` | Come in [consultare i documenti per spedizione](#endpoint-consultare-i-documenti-per-spedizione) |
| `base64` | query | boolean | no | - | `true` | Come in [consultare i documenti per spedizione](#endpoint-consultare-i-documenti-per-spedizione) |
| `documents` | query | string | no | - | tutti | Come in [consultare i documenti per spedizione](#endpoint-consultare-i-documenti-per-spedizione) |

> **Come funziona la ricerca:**
>
> - **Con `channel_code`:** cerca la quotazione più recente con questo identificatore **e** questo codice di canale.
> - **Senza `channel_code`:** cerca la quotazione più recente con questo identificatore, **con o senza** codice di canale.
>   Se usi lo stesso identificatore in canali diversi, invia sempre il `channel_code`.
> - A differenza della [consultazione dello status per canale](api-rastreio.md#endpoint-consultare-lo-status-per-canale), questa ricerca
>   **considera anche le spedizioni annullate**.
> - Per omettere il `channel_code`, è sufficiente non inviare il segmento. Anche il valore `,` viene trattato come assente.

### Payload minimo

```
GET /v1/shipping/docs/PED-2026-0001
```

### Payload massimo

```
GET /v1/shipping/docs/PED-2026-0001/1?attachments=true&embedded=true&base64=true&documents=awb_pdf,awb_zpl,awb_prc_pdf,awb_prc_zpl,awb_prc_fedex_pdf,awb_prc_fedex_zpl,invoice_pdf,packing_list_pdf,declaration_pdf,non_dangerous_pdf
```

### Esempio di request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/docs/PED-2026-0001/1?base64=false' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response

`200 OK` — stessa struttura di [consultare i documenti per spedizione](#endpoint-consultare-i-documenti-per-spedizione).

`422 Unprocessable Entity` — `channel_code` non numerico:

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "channel_code": [
      "O campo channel code deve conter um número inteiro."
    ]
  }
}
```

### Campi della risposta

Identici a quelli di [consultare i documenti per spedizione](#campi-della-risposta).

## Regole di business
- I documenti esistono solo **dopo** l'elaborazione presso il corriere (gruppo `shipped` nel tracciamento).
- **Ordine di decisione della risposta:**

  | Documenti trovati (dopo il filtro) | Allegati (`attachments=true` e `base64=true`) | Risposta |
  |---|---|---|
  | Sì | Sì o no | `200` con `docs`, `attachments`, `embedded_docs`, `carrier` e `tracking_id` |
  | No | Sì | `200` con `docs: []`, `attachments` e `embedded_docs: []` (senza `carrier` e `tracking_id`) |
  | No | No | `400` `Nenhum documento encontrado com o código especificado!` |

- Il filtro `documents` vale anche per la decisione precedente: se i tipi filtrati non esistono ancora e non ci sono
  allegati, la risposta è `400`.
- I documenti il cui file non è disponibile vengono omessi silenziosamente. Se non ne rimane nessuno e non ci sono
  allegati, la risposta è `400`.
- `embedded_docs` dipende da un **permesso specifico** del tuo account. Senza di esso, la lista arriva vuota, anche con
  `embedded=true`.
- **Validità degli URL:** quando il tuo account è configurato per proteggere i link dei documenti, ogni `url`
  è **temporanea e vale 5 minuti**. Negli altri casi il link è pubblico, ma può anche essere temporaneo.
  **Scarica e conserva** i file dalla tua parte subito dopo la consultazione; per scaricarli di nuovo in seguito, consulta di nuovo
  l'endpoint.
- Vengono restituiti solo i documenti delle spedizioni del tuo account. Le spedizioni di un altro account rispondono
  `400 Nenhum documento encontrado com o código especificado!` (per UUID e per canale) oppure
  `404 Esta remessa não foi encontrada!` (per AWB e per NF-e).
- La ricerca per AWB considera solo le spedizioni create negli **ultimi 90 giorni**.
- La ricerca per chiave della NF-e esclude le spedizioni annullate.

## Errori comuni

| Status | Significato/Messaggio | Come correggere |
|---|---|---|
| `400` | `Nenhum documento encontrado com o código especificado!` | Attendere l'elaborazione (status `shipped`), rivedere il filtro `documents` o verificare l'identificatore. Nella consultazione per canale, indica anche che non è stata trovata nessuna spedizione |
| `400` | `Remessa não encontrada!` — nessuna spedizione con l'UUID indicato (consultazione per `{shipping}`) | Verificare l'UUID della quotazione |
| `403` | `Has no permission!` | Richiedere al supporto il permesso di consultazione dei documenti |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Account bloccato: contattare il supporto |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificare il token inviato in `Authorization` |
| `404` | `Esta remessa não foi encontrada!` — AWB o chiave della NF-e senza spedizione corrispondente nel tuo account (AWB con più di 90 giorni, NF-e di una spedizione annullata) | Verificare l'AWB/NF-e o usare l'UUID della quotazione |
| `422` | `O campo channel code deve conter um número inteiro.` (in `data.channel_code`) | Inviare un intero in `channel_code` oppure omettere il segmento |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar documentos da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Ripetere con backoff; se persiste, contattare il supporto |

## Note finali
- La consultazione per `{shipping}` con l'UUID della quotazione è il modo più diretto e stabile.
- La consultazione per canale è l'opzione migliore quando vuoi lavorare solo con il numero del tuo ordine.
- Combina `documents` + `base64=false` per risposte rapide nelle integrazioni ad alto volume.
- Per sapere quando i documenti diventano disponibili, vedi [Tracciamento della Spedizione](api-rastreio.md).
- Per inviare e rimuovere allegati, vedi [Spedizione (v1)](api-remessa-v1.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
