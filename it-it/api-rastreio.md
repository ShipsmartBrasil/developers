# Tracciamento della Spedizione

- [Torna al README della lingua](README.md)

## Panoramica
- Consulta lo status operativo e lo storico di una spedizione.
- Mostra ricezione, pagamento, processamento, trasporto e annullamento.
- Associato al flusso operativo principale.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header della lingua: `lang`.

## Convenzioni
- Il campo `shipping` accetta UUID, AWB o NF-e, secondo l'endpoint.
- Gli status usano codici interni come `RCV0001`, `PPY0001`, `SPD0001`, `CCG0001` e `CCD0001`.

## Formati di risposta
- Lo stesso endpoint restituisce due formati: `timeline` e `status`.
- `timeline` porta gli eventi nell'ordine in cui arrivano dal vettore.
- `status` porta gli eventi raggruppati per status, usando la mappatura interna della piattaforma per semplificare la lettura.

## Flusso di utilizzo
1. Crea la spedizione.
2. Consulta lo status tramite `shipping` o tramite canale.
3. Monitora `received`, `processing_payment`, `processing_shipment`, `shipped` e, se applicabile, annullamento.

## Endpoints
### Status per spedizione
- Metodo e rotta: `GET /v1/shipping/{shipping}/status`

### Status per canale
- Metodo e rotta: `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | sì | Identificatore della spedizione | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Accetta UUID, AWB e NF-e |
| `channel_identifier_number` | path | string | sì | Numero del canale | `334455` | - |
| `channel_code` | path | integer | no | Codice del canale | `1` | Opzionale |

### Esempio di request
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response
```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Fedex Economy Startup Test BR",
      "tracking_number": "271249639538",
      "timeline": [
        {
          "update_date": "2024-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        }
      ]
    },
    "received": [],
    "processing_payment": [],
    "shipped": []
  }
}
```

## Regole di business
- Il pagamento deve esistere affinché la spedizione restituisca status completo.
- Se il pagamento non è ancora stato approvato, lo status può restare in `processing_payment`.
- Quando la spedizione è fatturata e non ancora processata dal vettore, può apparire `processing_shipment`.
- Quando c'è annullamento, la risposta può includere `cancelling` o `cancelled`.

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `400` | Spedizione non trovata | Verificare identificatore |
| `404` | Senza pagamento collegato | Confermare processamento della spedizione |
| `500` | Errore imprevisto | Contattare il team responsabile |

## Osservazioni finali
- Il tracciamento è la forma più completa di monitorare il ciclo di vita dell'invio.
- La timeline deriva dagli eventi interni e dal ritorno del pagamento/vettore.
- Il raggruppamento di status è un riepilogo operativo per facilitare la comprensione del tracciamento della spedizione.
