# Documenti di Spedizione

- [Torna al README della lingua](README.md)

## Panoramica
- Consulta documenti generati dopo il processamento della spedizione.
- Può restituire PDF, ZPL, allegati e, quando consentito, documenti incorporati.
- Questo modulo è pubblicato nella v1.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header della lingua: `lang`.

## Convenzioni
- `attachments`, `embedded` e `base64` accettano booleani tramite query string.
- `documents` riceve lista separata da virgola.

## Flusso di utilizzo
1. Crea la spedizione.
2. Attendi che lo status indichi processamento/conclusione.
3. Consulta i documenti per `shipping` o per canale.
4. Scarica il contenuto in `url` o `base64`.

## Endpoints
### Consultare documenti per spedizione
- Metodo e rotta: `GET /v1/shipping/{shipping}/docs`
- `shipping` accetta UUID della quotazione, AWB o NF-e.

### Consultare documenti per canale
- Metodo e rotta: `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | sì | Identificatore della spedizione | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Accetta anche AWB e NF-e |
| `channel_identifier_number` | path | string | sì | Identificatore del canale | `334455` | - |
| `channel_code` | path | integer | no | Codice del canale | `1` | Opzionale |
| `attachments` | query | boolean | no | Include allegati | `true` | Default `true` |
| `embedded` | query | boolean | no | Include docs incorporati | `true` | Default `true` |
| `base64` | query | boolean | no | Include base64 | `true` | Default `true` |
| `documents` | query | string | no | Filtra documenti | `invoice_pdf,awb_pdf` | Quando vuoto, restituisce tutto |

### Esempio di request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response
```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "carrier": "DHL - Teste",
    "tracking_id": "000000",
    "docs": {
      "Invoice": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "string"
      },
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    },
    "embedded_docs": [
      { "type": "pdf", "base64": "string" }
    ],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "string" }
    ]
  }
}
```

## Regole di business
- Se non ci sono documenti né allegati, l'API può restituire `warning`.
- I documenti incorporati dipendono da permesso specifico.
- I tipi di documenti filtrabili sono quelli elencati nella risposta della spedizione.

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `400` | Nessun documento trovato | Confermare se la spedizione è già stata processata |
| `500` | Errore imprevisto | Ripetere e contattare il team responsabile |

## Osservazioni finali
- Per integrazioni, la consultazione per `shipping` è la forma più diretta.
- La consultazione per canale è utile quando l'integratore registra l'identificatore commerciale nell'invio.
