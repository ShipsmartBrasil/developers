# Tracciamento della Spedizione

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Consulta lo status operativo e lo storico completo di una spedizione, dalla ricezione sulla piattaforma fino alla
  consegna (o all'annullamento).
- La stessa risposta contiene **due formati**: una `timeline` cronologica e gli eventi **raggruppati per status**.
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

> Le descrizioni di status (`status_description`) sono tradotte in base all'header `lang`.
> I codici (`status_code`) **non** cambiano con la lingua — usali per la logica di integrazione.

## Endpoint

### Status per spedizione
- **Metodo e rotta:** `GET /v1/shipping/{shipping}/status`

### Status per canale
- **Metodo e rotta:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

## Parametri

| Nome | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `shipping` | path | string | **sì** | UUID della quotazione, numero dell'AWB (ultimi 90 giorni) o chiave della NF-e (44 cifre) |
| `channel_identifier_number` | path | string | **sì** (variante per canale) | Identificatore commerciale indicato nella quotazione |
| `channel_code` | path | integer | no | Codice del canale di vendita |

## Esempio di request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

Per canale:
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/status/PED-2026-0001/1 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

## Esempio di response

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
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:40",
          "status_code": "POT0001",
          "status_description": "Pagamento aprovado junto ao gateway: approved"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:44:38",
          "status_code": "SPD0001",
          "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
        },
        {
          "update_date": "2026-02-22",
          "update_time": "09:10:02",
          "status_code": "CLD0001",
          "status_description": "Remessa coletada pelo transportador"
        },
        {
          "update_date": "2026-02-23",
          "update_time": "04:35:11",
          "status_code": "ITR0001",
          "status_description": "Remessa em trânsito internacional"
        }
      ]
    },
    "cost_center": {
      "code": 1,
      "name": "E-commerce"
    },
    "received": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:38",
        "status_code": "RCV0001",
        "status_description": "Remessa foi recebida em nosso sistema"
      }
    ],
    "paid_out": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:40",
        "status_code": "POT0001",
        "status_description": "Pagamento aprovado junto ao gateway: approved"
      }
    ],
    "shipped": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:44:38",
        "status_code": "SPD0001",
        "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
      }
    ],
    "collected": [
      {
        "update_date": "2026-02-22",
        "update_time": "09:10:02",
        "status_code": "CLD0001",
        "status_description": "Remessa coletada pelo transportador"
      }
    ],
    "in_transit": [
      {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    ],
    "last_update": {
      "in_transit": {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    }
  }
}
```

## Struttura della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.carrier.name` | string | Nome del contratto/corriere che ha elaborato la spedizione |
| `data.carrier.tracking_number` | string | Numero di tracciamento (AWB) presso il corriere |
| `data.carrier.timeline` | array | **Tutti gli eventi in ordine cronologico.** Usa questo campo per costruire una linea del tempo nella tua interfaccia |
| `data.cost_center` | object | **Opzionale.** Compare solo quando la spedizione ha un centro di costo |
| `data.cost_center.code` | integer | Codice del centro di costo |
| `data.cost_center.name` | string | Nome del centro di costo |
| `data.<gruppo_di_status>` | array | Eventi raggruppati per status. Le chiavi presenti variano in base all'avanzamento |
| `data.last_update` | object | **Ultimo evento**, nel formato `{ "<gruppo>": { ...evento } }`. Usalo per conoscere lo stato attuale con una sola lettura |

Ogni evento (nella `timeline`, nei gruppi e in `last_update`) ha la stessa forma:

| Campo | Tipo | Descrizione |
|---|---|---|
| `update_date` | string | Data dell'evento (`Y-m-d`) |
| `update_time` | string | Ora dell'evento (`H:i:s`) |
| `status_code` | string | Codice interno dello status (vedi tabella) o il codice grezzo del gateway/corriere |
| `status_description` | string | Descrizione leggibile, tradotta dall'header `lang` |

### I due formati, e quando usare ciascuno

| Formato | Dove si trova | Usalo per |
|---|---|---|
| Cronologico | `data.carrier.timeline` | Mostrare lo storico completo all'utente finale |
| Raggruppato per status | `data.received`, `data.shipped`, `data.in_transit`... | Verificare se la spedizione **è già passata** per una determinata fase (`if (isset(data.shipped))`) |
| Ultimo stato | `data.last_update` | Dashboard e sincronizzazioni: una lettura risolve "dov'è la spedizione" |

## Gruppi di status e codici

| Gruppo (chiave in `data`) | `status_code` | Significato |
|---|---|---|
| `received` | `RCV0001` | La spedizione è stata ricevuta sulla piattaforma |
| `processing_payment` | `PPY0001` | In attesa della conferma del pagamento nel gateway |
| `paid_out` | `POT0001` | Pagamento approvato |
| `pending_consolidation` | `PCD0001` | In attesa di consolidamento (contratti con consolidamento) |
| `consolidating` | `CDT0001` | In consolidamento |
| `processing_shipment` | `PSH0001` | Pagamento risolto, in attesa dell'elaborazione presso il corriere |
| `shipped` | `SPD0001` | **Elaborata presso il corriere. Documenti disponibili** |
| `collecting` | `CLG0001` | Ritiro in corso |
| `collected` | `CLD0001` | Ritirata dal corriere |
| `in_transit` | `ITR0001` | In transito |
| `delivering` | `DLG0001` | In consegna |
| `delivered` | `DLD0001` | Consegnata |
| `error` | `ERR0001` | Errore nell'elaborazione |
| `cancelling` | `CCG0001` | Annullamento richiesto, in elaborazione |
| `cancelled` | `CCD0001` | Annullata |
| `foc` | `FOC0001` | Finalizzata dal corriere |
| `eol` | `EOL0001` | Fine del ciclo di vita del tracciamento |

> **`status_code` può contenere il codice grezzo del gateway o del corriere** quando esiste
> (es.: `approved`, `charge.succeeded`, codici propri del corriere). I codici `XXX0001` sono la
> corrispondenza interna della piattaforma, usata quando non c'è un codice esterno. Per la logica di business, preferisci la
> **chiave del gruppo** (`shipped`, `delivered`, `cancelled`...) invece dello `status_code`.

## Flusso tipico degli stati

```
received
   │
   ├── processing_payment ──► paid_out        (pagamento via gateway)
   │
   └── (pagamento billed: prosegue diretto)
                │
                ├── pending_consolidation ──► consolidating   (quando c'è consolidamento)
                │
                ▼
         processing_shipment
                │
                ▼
             shipped          ◄── documenti disponibili da qui in poi
                │
                ▼
    collecting ──► collected ──► in_transit ──► delivering ──► delivered
                │
                └── error | cancelling ──► cancelled | foc | eol
```

## Regole di business

- **Ogni spedizione deve avere una registrazione di pagamento** affinché lo status possa essere costruito. Se non c'è,
  la risposta è `404` con `Nenhuma entrada de pagamento foi registrada para esta remessa!`.
- Con `payment.type: billed`, la spedizione non passa per `processing_payment`: prosegue diretta verso
  `processing_shipment`/`shipped`.
- Con `payment.type: gateway`, la spedizione resta in `processing_payment` fino all'approvazione del gateway; solo allora
  avanza verso `processing_shipment`.
- Il gruppo passa da `processing_shipment` a `shipped` nel momento in cui i **documenti vengono generati**.
  È questo il segnale per chiamare [Documenti della Spedizione](api-documentos.md).
- Quando la spedizione è in `cancelling`, `cancelled` o `error`, il `last_update` usa la descrizione propria
  della piattaforma invece del testo grezzo del corriere.
- Le spedizioni che restano **29 giorni o più** senza eventi di tracciamento in uno status eleggibile possono essere
  annullate automaticamente dalla piattaforma.
- La ricerca per AWB funziona per le spedizioni create negli **ultimi 90 giorni**.

## Buone pratiche di polling

| Situazione | Intervallo consigliato |
|---|---|
| Subito dopo la creazione della spedizione (in attesa di `shipped`) | Da 10 a 30 secondi, con backoff |
| Spedizione già in `shipped`/`in_transit` | Da 1 a 4 ore |
| Spedizione in `delivered`, `cancelled`, `foc` o `eol` | Interrompi il polling: sono stati finali |

1. Leggi `data.last_update` per decidere se c'è stato un cambiamento — è la lettura più economica.
2. Interrompi il polling negli stati finali.
3. Usa un backoff esponenziale sui `500`.
4. Tratta `400`/`404` come condizione di business: non ripetere la stessa richiesta in loop.

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Verificare `{shipping}` o l'identificatore del canale |
| `403` | Permesso mancante per consultare lo status | Richiedere il permesso al supporto |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Attendere la registrazione del pagamento; se persiste, contattare il supporto |
| `422` | `channel_code` non numerico | Inviare un intero in `channel_code` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- `data.carrier.timeline` è la vista più completa; i gruppi sono un riepilogo operativo per facilitare la
  lettura programmatica.
- Conserva il `tracking_number` per offrire al cliente finale il tracciamento sul sito del corriere.
- Per scaricare etichetta e invoice appena la spedizione raggiunge `shipped`, vedi
  [Documenti della Spedizione](api-documentos.md).
