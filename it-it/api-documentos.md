# Documenti della Spedizione

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Recupera i documenti generati dalla piattaforma e dal corriere dopo l'elaborazione della spedizione:
  AWB (etichetta), invoice, packing list, dichiarazione e altri.
- Restituisce anche gli **allegati** che hai inviato alla creazione/aggiornamento della spedizione.
- Ogni documento arriva con una `url` per il download e, facoltativamente, con il contenuto in `base64`.
- Funziona per le spedizioni create in **v1 e in v2**.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Documentazione interattiva: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticazione
```
Authorization: Bearer IL_TUO_TOKEN
lang: pt
```

## Flusso di utilizzo
1. Crea la [spedizione](api-remessa-v1.md).
2. Consulta lo [status](api-rastreio.md) fino a quando la spedizione compare nel gruppo `shipped` — è in quel momento che
   i documenti iniziano a esistere.
3. Consulta i documenti tramite `{shipping}` o tramite il canale.
4. Scarica il file dalla `url` (consigliato) o decodifica il `base64`.

> **Prima di `shipped` la consultazione può restituire `400 Nenhum documento encontrado com o código especificado!`.**
> È un comportamento previsto: attendi l'elaborazione presso il corriere.

## Endpoint

### Consultare i documenti per spedizione
- **Metodo e rotta:** `GET /v1/shipping/{shipping}/docs`

### Consultare i documenti per canale
- **Metodo e rotta:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`
- Utile quando hai conservato solo il numero del tuo stesso ordine.

## Parametri

| Nome | Posizione | Tipo | Obbligatorio | Predefinito | Descrizione |
|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | UUID della quotazione, numero dell'AWB (ultimi 90 giorni) o chiave della NF-e (44 cifre) |
| `channel_identifier_number` | path | string | **sì** (variante per canale) | - | Identificatore commerciale indicato nella quotazione |
| `channel_code` | path | integer | no | - | Codice del canale di vendita |
| `attachments` | query | boolean | no | `true` | Includere gli allegati inviati da te |
| `embedded` | query | boolean | no | `true` | Includere i documenti incorporati (AWB + invoice consolidati in un unico file). Richiede un permesso specifico |
| `base64` | query | boolean | no | `true` | Includere il contenuto del file in Base64, oltre alla `url` |
| `documents` | query | string | no | tutti | Elenco di tipi separati da virgole. Quando è vuoto o assente, restituisce tutti |

> **Suggerimento sulle prestazioni:** invia `base64=false` quando scarichi dalla `url`. Il payload diventa molto più piccolo
> e la risposta molto più rapida.

## Tipi di documento filtrabili (`documents`)

| Valore in `documents` | Chiave in `data.docs` | Contenuto |
|---|---|---|
| `awb_pdf` | `AWB` | Etichetta/AWB in PDF |
| `awb_zpl` | `AWB_ZPL` | Etichetta/AWB in ZPL (stampante termica) |
| `awb_prc_pdf` | `AWB PRC` | AWB del processo di reso/partner in PDF |
| `awb_prc_zpl` | `AWB PRC ZPL` | AWB del processo di reso/partner in ZPL |
| `awb_prc_fedex_pdf` | `AWB PRC Fedex` | AWB FedEx del processo di reso/partner in PDF |
| `awb_prc_fedex_zpl` | `AWB PRC Fedex ZPL` | AWB FedEx del processo di reso/partner in ZPL |
| `invoice_pdf` | `Invoice` | Commercial Invoice in PDF |
| `packing_list_pdf` | `Packing List` | Packing list in PDF |
| `declaration_pdf` | `Declaration` | Dichiarazione di contenuto in PDF |
| `non_dangerous_pdf` | `Non-Dangerous Object` | Dichiarazione di merce non pericolosa in PDF |

Esempio di filtro:
```
?documents=invoice_pdf,awb_pdf,awb_zpl
```

> Non tutti i documenti esistono in ogni spedizione: l'insieme dipende dal corriere, dal paese di destinazione e dal
> tipo di oggetto. In `data.docs` compaiono solo i documenti effettivamente generati.

## Esempio di request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=false&base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

Per canale:
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/docs/PED-2026-0001/1?base64=false' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

## Esempio di response

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "carrier": "DHL - Test",
    "tracking_id": "1234567890123",
    "docs": {
      "Invoice": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      },
      "Packing List": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_packing_list.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Declaration": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_declaration.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Non-Dangerous Object": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_non_dangerous_object.pdf",
        "base64": "JVBERi0xLjcK..."
      }
    },
    "embedded_docs": [
      { "type": "pdf", "base64": "JVBERi0xLjcK..." },
      { "type": "zpl", "base64": "XlhBDQpeTUNZ..." }
    ],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Fattura", "base64": "JVBERi0xLjcK..." },
      { "id": 2, "type": "xml", "description": "XML della Fattura", "base64": "PD94bWwg..." }
    ]
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.carrier` | string | Nome del corriere che ha elaborato la spedizione |
| `data.tracking_id` | string | Numero di tracciamento/AWB |
| `data.docs` | object | Mappa `nome del documento → { url, base64 }`. Le chiavi sono quelle della tabella sopra |
| `data.docs.*.url` | string | URL per il download diretto del file |
| `data.docs.*.base64` | string | Contenuto in Base64. Presente solo quando `base64=true` |
| `data.embedded_docs` | array | Documenti incorporati. `null`/assente quando `embedded=false` o senza permesso |
| `data.embedded_docs[].type` | string | `pdf` o `zpl` |
| `data.embedded_docs[].base64` | string | Contenuto in Base64 |
| `data.attachments` | array | Allegati inviati da te. `null`/assente quando `attachments=false` |
| `data.attachments[].id` | integer | **ID dell'allegato.** Usalo per rimuoverlo con `DELETE /v1/shipping/{shipping}/docs` |
| `data.attachments[].type` | string | `pdf`, `xml`, `jpeg` o `png` |
| `data.attachments[].description` | string | Descrizione indicata alla creazione |
| `data.attachments[].base64` | string | Contenuto in Base64 |

## Stampare le etichette

| Formato | Quando usarlo |
|---|---|
| `awb_pdf` | Stampante comune (A4/Letter). Apri la `url` o decodifica il `base64` |
| `awb_zpl` | Stampante termica Zebra. Invia il contenuto **puro** (dopo aver decodificato il Base64) diretto alla coda della stampante |

> Il contenuto ZPL è testo. Non convertirlo né rielaborarlo: invialo esattamente come è arrivato dopo il decode.

## Gestire gli allegati

| Operazione | Rotta | Pagina |
|---|---|---|
| Inviare allegati alla creazione | `POST /v1/shipping` (campo `attachments`) | [Spedizione (v1)](api-remessa-v1.md) |
| Inviare allegati in seguito | `PUT /v1/shipping/{shipping}` | [Spedizione (v1)](api-remessa-v1.md#endpoint-aggiornare-spedizione) |
| Elencare gli allegati | `GET /v1/shipping/{shipping}/docs?attachments=true` | Questa pagina |
| Rimuovere allegati | `DELETE /v1/shipping/{shipping}/docs` | [Spedizione (v1)](api-remessa-v1.md#endpoint-rimuovere-allegati) |

Limiti degli allegati:

| Limite | Valore |
|---|---|
| Formati accettati | `PDF`, `XML`, `JPEG`, `PNG` |
| Dimensione per file | **5 MB** |
| Dimensione totale della richiesta | **90 MB** |

## Regole di business
- I documenti esistono solo **dopo** l'elaborazione presso il corriere (gruppo `shipped` nel tracciamento).
- `embedded_docs` dipende da un **permesso specifico** sul tuo account. Senza di esso, il campo non viene restituito,
  anche con `embedded=true`. Messaggio correlato:
  `Para obter os documentos incoporados a remessa precisa ter sido ao menos enviada.`
- Se non c'è alcun documento né allegato, la risposta è `400` con
  `Nenhum documento encontrado com o código especificado!`.
- Le `url` puntano a uno storage esterno e possono avere una validità limitata. **Scarica e archivia** i file
  dalla tua parte se devi conservarli a lungo.
- La consultazione per canale considera il `channel_identifier_number` indicato nella **quotazione**.
  Senza `channel_code`, la ricerca usa solo l'identificatore.

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | `Nenhum documento encontrado com o código especificado!` | Attendere l'elaborazione (status `shipped`) o verificare l'identificatore |
| `403` | Permesso mancante per consultare i documenti | Richiedere il permesso al supporto |
| `404` | Spedizione non trovata | Verificare `{shipping}` o l'identificatore del canale |
| `422` | `channel_code` non numerico | Inviare un intero in `channel_code` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- La consultazione tramite `{shipping}` è la forma più diretta e stabile.
- La consultazione per canale è l'opzione migliore quando vuoi lavorare solo con il numero del tuo ordine.
- Combina `documents` + `base64=false` per risposte rapide nelle integrazioni ad alto volume.
