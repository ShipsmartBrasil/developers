# Tracciamento della Spedizione

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Consulta lo status operativo e lo storico di una spedizione, dalla ricezione sulla piattaforma fino alla
  consegna (o all'annullamento).
- La stessa risposta offre **tre letture**: la `timeline` cronologica, gli eventi **raggruppati per status** e
  l'**ultimo stato** (`last_update`).
- Ci sono due modi di consultare: tramite l'identificatore della spedizione (`{shipping}`) o tramite l'identificatore del tuo
  canale di vendita (`channel_identifier_number` + `channel_code`).
- Funziona per le spedizioni create in **v1 e in v2** (le rotte di tracciamento esistono solo nella v1).

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
| `Authorization` | **sì** | `Bearer <token>` | - | Token del tuo account. Il profilo deve avere il permesso di consultazione dello status della spedizione |
| `lang` | no | `pt`, `en`, `es` | `pt` | Lingua dei messaggi e delle descrizioni generate dalla piattaforma |

> L'header `lang` traduce il `message` della risposta e le descrizioni degli eventi generati dalla **piattaforma**
> (ricezione, pagamento, elaborazione, annullamento ed errore). Le descrizioni degli eventi provenienti dal
> **corriere** vengono restituite così come il corriere le ha inviate e **non** cambiano con la lingua.
> I codici (`status_code`) e le chiavi dei gruppi non cambiano mai con la lingua — usali nella logica dell'integrazione.

## Flusso di utilizzo
1. Crea la spedizione ([Spedizione v1](api-remessa-v1.md) o [Spedizione v2](api-remessa-v2.md)) e conserva il `quotation`
   (UUID) usato nella creazione.
2. Consulta lo status tramite `{shipping}` (consigliato) o tramite l'identificatore del canale.
3. Quando compare il gruppo `shipped`, i documenti esistono già: scaricali in [Documenti della Spedizione](api-documentos.md).
4. Continua a consultare a intervalli più lunghi per seguire ritiro, transito e consegna.
5. Smetti di consultare quando `last_update` indica uno stato finale (vedi [Buone pratiche di polling](#buone-pratiche-di-polling)).

```
crea spedizione ──► GET .../status ──► (shipped?) ──► GET .../docs
                          │                 │
                          │ no              │ sì
                          ▼                 ▼
                   attendi e ripeti    segui ritiro → transito → consegna
```

## Concetti essenziali

### Le tre letture della risposta

| Lettura | Dove si trova | Cosa contiene | Da usare per |
|---|---|---|---|
| Cronologica | `data.carrier.timeline` | Tutti gli eventi costruiti dalla piattaforma e tutti gli eventi di tracciamento del corriere, nell'ordine in cui sono stati registrati | Mostrare lo storico completo all'utente finale |
| Raggruppata per status | `data.received`, `data.shipped`, `data.in_transit`... | Una chiave per ogni status già raggiunto. Ogni chiave è una lista con **un solo evento**: il più recente di quello status | Verificare se la spedizione **è già passata** per una determinata fase |
| Ultimo stato | `data.last_update` | Lo stato attuale, nel formato `{ "<grupo>": { ...evento } }` | Dashboard e sincronizzazioni: una sola lettura risolve "dove si trova la spedizione" |

> **Attenzione:** i gruppi non conservano lo storico. Se il corriere invia tre eventi `in_transit`, il gruppo
> `data.in_transit` riporterà solo l'ultimo; tutti e tre compaiono in `data.carrier.timeline`.

### Gruppi di status e codici

| Gruppo (chiave) | `status_code` della piattaforma | Significato | Dove può comparire |
|---|---|---|---|
| `received` | `RCV0001` | La spedizione è stata ricevuta sulla piattaforma | Gruppo e `timeline` (sempre presente) |
| `processing_payment` | `PPY0001` | Pagamento non ancora approvato (è anche l'evento mostrato per il pagamento fatturato, `billed`) | Gruppo e `timeline` |
| `paid_out` | `POT0001` | Pagamento approvato dal mezzo di pagamento | Gruppo e `timeline` |
| `pending_consolidation` | `PCD0001` | In attesa di consolidamento (contratti con consolidamento) | Riservato (vedi [Regole di business](#regole-di-business)) |
| `consolidating` | `CDT0001` | In consolidamento | Riservato (vedi [Regole di business](#regole-di-business)) |
| `processing_shipment` | `PSH0001` | Pagamento risolto, in attesa dell'elaborazione presso il corriere | Gruppo e `timeline` |
| `shipped` | `SPD0001` | **Elaborata presso il corriere. Documenti disponibili** | Gruppo e `timeline` |
| `collecting` | `CLG0001` | Corriere in viaggio per il ritiro | Gruppo, `timeline` e `last_update` |
| `collected` | `CLD0001` | Ritirata dal corriere | Gruppo, `timeline` e `last_update` |
| `in_transit` | `ITR0001` | In transito | Gruppo, `timeline` e `last_update` |
| `delivering` | `DLG0001` | In consegna | Gruppo, `timeline` e `last_update` |
| `delivered` | `DLD0001` | Consegnata | Gruppo, `timeline` e `last_update` |
| `error` | `ERR0001` | Non è stato possibile completare la spedizione | Solo `last_update` |
| `cancelling` | `CCG0001` | Annullamento richiesto, in elaborazione | Solo `last_update` |
| `cancelled` | `CCD0001` | Annullata | Solo `last_update` |
| `foc` | `FOC0001` | Conclusa dal corriere | Solo `last_update` |
| `eol` | `EOL0001` | Fine del ciclo di vita del tracciamento | Solo `last_update` |

> **`status_code` non è sempre un codice `XXX0001`.** Riporta il codice grezzo dell'origine dell'evento quando
> tale codice esiste:
>
> - nell'evento di pagamento: lo status indicato dal mezzo di pagamento (es.: `approved`);
> - negli eventi di tracciamento: il codice proprio del corriere.
>
> I codici `XXX0001` vengono usati quando non esiste un codice esterno. Per la logica di business, usa la **chiave del
> gruppo** (`shipped`, `in_transit`, `delivered`, `cancelled`...) invece dello `status_code`.

### Descrizioni generate dalla piattaforma

Messaggi letterali in `pt` (cambiano in base a `lang`):

| Gruppo | `status_description` |
|---|---|
| `received` | `Remessa foi recebida em nosso sistema` |
| `processing_payment` | `Aguardando a finalização do processamento do pagamento junto ao gateway: :description ` |
| `paid_out` | `Pagamento aprovado junto ao gateway: :description ` |
| `processing_shipment` | `Remessa esta sendo processada no transportador especificado!` |
| `shipped` | `Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!` |
| `cancelling` (in `last_update`) | `Cancelando Remessa` |
| `cancelled` (in `last_update`) | `Remessa Cancelada` |
| `error` (in `last_update`) | `Não foi possível concluir o embarque desta remessa.` |

> Nei messaggi di pagamento, `:description` è la descrizione inviata dal mezzo di pagamento oppure, in sua mancanza,
> il codice stesso della piattaforma (`PPY0001` / `POT0001`). Entrambi i messaggi terminano con uno **spazio** —
> tienine conto se confronti stringhe (si consiglia di non confrontare le descrizioni).

### Flusso tipico degli stati

```
received
   │
   ├── processing_payment ──► paid_out          (pagamento tramite gateway)
   │
   ├── processing_payment                       (pagamento billed: prosegue diretto)
   │
   ▼
processing_shipment
   │
   ▼
shipped                 ◄── documenti disponibili da qui in poi
   │
   ▼
collecting ──► collected ──► in_transit ──► delivering ──► delivered

Stati mostrati solo in last_update:
   error | cancelling ──► cancelled | foc | eol
```

## Endpoint: consultare lo status per spedizione

- **Metodo e rotta:** `GET /v1/shipping/{shipping}/status`
- **Quando usarlo:** è il modo consigliato. Usa l'UUID della quotazione che hai inviato come `quotation` nella creazione della spedizione.

### Parametri

| Nome | Posizione | Tipo | Obbligatorio | Dimensione | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | - | **Forma A:** UUID della quotazione usata nella spedizione. **Forma B:** numero di tracciamento (AWB) della spedizione |

> **Identificatori accettati in questa rotta:** UUID della quotazione o AWB. Se l'AWB è presente in più di una spedizione,
> viene considerata la più recente. **La chiave della NF-e non è accettata in questa rotta** (la consultazione risponde
> `400`), sebbene sia accettata nella rotta dei documenti.

### Payload minimo

Endpoint senza corpo. È sufficiente il parametro di path:

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status
```

### Payload massimo

Non ci sono parametri di query. La forma massima è uguale alla minima, cambia solo l'identificatore:

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status
GET /v1/shipping/1234567890123/status
```

### Esempio di request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response

`200 OK` — spedizione con pagamento fatturato (`billed`), già elaborata e in transito:

```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Contratto Esempio Espresso",
      "tracking_number": "1234567890123",
      "timeline": [
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "PPY0001",
          "status_description": "Aguardando a finalização do processamento do pagamento junto ao gateway: PPY0001 "
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
          "status_code": "PU",
          "status_description": "Shipment picked up"
        },
        {
          "update_date": "2026-02-23",
          "update_time": "04:35:11",
          "status_code": "IT",
          "status_description": "In transit"
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
    "processing_payment": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:38",
        "status_code": "PPY0001",
        "status_description": "Aguardando a finalização do processamento do pagamento junto ao gateway: PPY0001 "
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
        "status_code": "PU",
        "status_description": "Shipment picked up"
      }
    ],
    "in_transit": [
      {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "IT",
        "status_description": "In transit"
      }
    ],
    "last_update": {
      "in_transit": {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "IT",
        "status_description": "In transit"
      }
    }
  }
}
```

`200 OK` — estratto di `last_update` di una spedizione annullata (sostituisce l'ultimo evento del corriere):

```json
{
  "last_update": {
    "cancelled": {
      "update_date": "2026-02-24",
      "update_time": "10:00:00",
      "status_code": "CCD0001",
      "status_description": "Remessa Cancelada"
    }
  }
}
```

`400 Bad Request` — spedizione di un altro account:

```json
{
  "status": "warning",
  "message": "Nenhuma remessa encontrada com o código especificado!"
}
```

### Campi della risposta

| Campo | Tipo | Presenza | Descrizione |
|---|---|---|---|
| `status` | string | sempre | `success`, `warning` o `error` |
| `message` | string | sempre | Messaggio tradotto in base all'header `lang` |
| `data.carrier.name` | string | sempre | Nome del contratto che ha elaborato la spedizione oppure, in assenza di contratto, il nome del corriere |
| `data.carrier.tracking_number` | string \| null | sempre | Numero di tracciamento (AWB) presso il corriere. Può arrivare `null`/vuoto prima dell'elaborazione |
| `data.carrier.timeline` | array | sempre | Tutti gli eventi, in ordine di registrazione: ricezione, pagamento, elaborazione e, successivamente, gli eventi di tracciamento del corriere |
| `data.cost_center` | object | facoltativo | Compare solo quando la spedizione ha un centro di costo |
| `data.cost_center.code` | integer | facoltativo | Codice del centro di costo |
| `data.cost_center.name` | string \| null | facoltativo | Nome del centro di costo |
| `data.received` | array | sempre | Evento di ricezione (un elemento) |
| `data.processing_payment` o `data.paid_out` | array | sempre uno dei due | Evento di pagamento (un elemento) |
| `data.processing_shipment` o `data.shipped` | array | dopo il pagamento approvato o fatturato | Evento di elaborazione presso il corriere (un elemento) |
| `data.collecting`, `data.collected`, `data.in_transit`, `data.delivering`, `data.delivered` | array | in base al tracciamento | Ultimo evento del corriere in quello status (un elemento) |
| `data.last_update` | object | facoltativo | Stato attuale. **Compare solo** dopo il primo evento di tracciamento del corriere o quando la spedizione è in `cancelling`, `cancelled` o `error` |

Ogni evento (in `timeline`, nei gruppi e in `last_update`) ha la stessa forma:

| Campo | Tipo | Formato | Descrizione |
|---|---|---|---|
| `update_date` | string | `AAAA-MM-GG` | Data dell'evento |
| `update_time` | string | `HH:MM:SS` | Ora dell'evento. **Eccezione:** nell'evento `processing_shipment`, questo campo arriva attualmente con la data (`AAAA-MM-GG`) invece dell'ora |
| `status_code` | string | - | Codice della piattaforma (`XXX0001`) o codice grezzo del mezzo di pagamento/corriere |
| `status_description` | string | - | Descrizione leggibile. Tradotta in base a `lang` solo negli eventi generati dalla piattaforma |

## Endpoint: consultare lo status per canale

- **Metodo e rotta:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`
- **Quando usarlo:** quando hai conservato solo l'identificatore dell'ordine nel tuo canale di vendita (indicato nella quotazione),
  e non l'UUID della quotazione.

### Parametri

| Nome | Posizione | Tipo | Obbligatorio | Dimensione | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `channel_identifier_number` | path | string | **sì** | - | - | Identificatore dell'ordine nel canale, esattamente come indicato nella quotazione |
| `channel_code` | path | integer | no | - | assente | Codice del canale di vendita indicato nella quotazione. Deve essere un numero intero |

> **Come funziona la ricerca:**
>
> - **Con `channel_code`:** cerca la spedizione la cui quotazione ha questo identificatore **e** questo codice di canale.
> - **Senza `channel_code`:** cerca **solo** spedizioni la cui quotazione è stata creata **senza** codice di canale.
>   Se la tua quotazione aveva `channel_code`, devi inviarlo.
> - Le spedizioni **annullate non vengono considerate**. Se ci sono più spedizioni valide, vale la più recente.
> - Per omettere il `channel_code`, è sufficiente non inviare l'ultimo segmento. Anche il valore `,` viene trattato come assente.

### Payload minimo

```
GET /v1/shipping/status/PED-2026-0001
```

### Payload massimo

```
GET /v1/shipping/status/PED-2026-0001/1
```

### Esempio di request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/status/PED-2026-0001/1 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response

`200 OK` — stessa struttura di [consultare lo status per spedizione](#endpoint-consultare-lo-status-per-spedizione).

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

Identici a quelli di [consultare lo status per spedizione](#campi-della-risposta).

## Regole di business

- **Solo spedizioni del tuo account.** La spedizione deve essere stata creata dall'account del token. Le spedizioni di un altro
  account rispondono `400` con `Nenhuma remessa encontrada com o código especificado!`.
- **Ogni spedizione deve avere una registrazione di pagamento** affinché lo status venga costruito. In caso contrario,
  la risposta è `404` con `Nenhuma entrada de pagamento foi registrada para esta remessa!` (messaggio fisso,
  non cambia con `lang`).
- **Pagamento e gruppo mostrato:**

  | `payment.type` della spedizione | Evento di pagamento | Evento di elaborazione |
  |---|---|---|
  | `billed` | `processing_payment` con `PPY0001` | `processing_shipment` → `shipped` |
  | `gateway` | `processing_payment` (codice del mezzo di pagamento) fino all'approvazione; poi `paid_out` | Compare solo dopo `paid_out` |
  | `afterward` | `processing_payment` fino all'approvazione del pagamento | Compare solo dopo l'approvazione del pagamento |

  Con `afterward`, la spedizione può già avere documenti senza che compaia il gruppo `shipped`. In tal caso, consulta
  direttamente [Documenti della Spedizione](api-documentos.md).
- Il gruppo passa da `processing_shipment` a `shipped` nel momento in cui i **documenti vengono generati**; la data
  e l'ora di `shipped` sono quelle della generazione del primo documento.
- I gruppi di tracciamento (da `collecting` a `delivered`) compaiono solo dopo che il corriere inizia a inviare eventi.
- Quando la spedizione è in `cancelling`, `cancelled` o `error`, il `last_update` usa la descrizione propria
  della piattaforma (tabella [Descrizioni generate dalla piattaforma](#descrizioni-generate-dalla-piattaforma)) invece del testo
  grezzo del corriere.
- **Consolidamento:** mentre la spedizione è in attesa di consolidamento o in consolidamento, la consultazione dello status può
  rispondere `500`. Attendi il completamento del consolidamento e consulta di nuovo.
- **Annullamento automatico:** le spedizioni che, **29 giorni dopo la creazione**, non hanno ancora un numero di tracciamento presso il
  corriere (e non sono andate oltre `shipped`) possono essere annullate automaticamente dalla piattaforma.
- **Fine del ciclo di vita:** le spedizioni con più di **2 mesi** che non hanno raggiunto uno stato finale
  (`delivered`, `cancelled`, `foc` o `eol`) vengono chiuse dalla piattaforma e smettono di ricevere aggiornamenti.
- La ricerca per AWB in questa rotta considera la spedizione più recente con quel numero.
- La consultazione per canale **ignora le spedizioni annullate**: per seguire un annullamento, consulta tramite `{shipping}`.

### Buone pratiche di polling

| Situazione | Intervallo consigliato |
|---|---|
| Subito dopo aver creato la spedizione (in attesa di `shipped`) | Da 10 a 30 secondi, con backoff |
| Spedizione già in `shipped`/`in_transit` | Da 1 a 4 ore |
| `last_update` in `delivered`, `cancelled`, `foc` o `eol` | Interrompi il polling: sono stati finali |
| Spedizione con più di 2 mesi | Interrompi il polling: la spedizione non riceve più aggiornamenti |

1. Leggi `data.last_update` per decidere se c'è stato un cambiamento — è la lettura più semplice. Ricorda che esiste
   solo dopo il primo evento del corriere (o in caso di annullamento/errore); prima di allora, usa i gruppi.
2. Interrompi il polling negli stati finali.
3. Usa un backoff esponenziale in caso di `500`.
4. Tratta `400`/`404` come condizione di business: non ripetere la stessa richiesta in loop.

## Errori comuni

| Status | Significato/Messaggio | Come correggere |
|---|---|---|
| `400` | `remessa.status.status_not_found: ` — nessuna spedizione trovata né per UUID né per AWB (consultazione per `{shipping}`; il messaggio arriva in questo formato, senza traduzione). Si verifica anche inviando una chiave della NF-e | Verificare il `{shipping}`. Usare l'UUID della quotazione o l'AWB |
| `400` | `Nenhuma remessa encontrada com o código especificado!` — spedizione di un altro account (per `{shipping}`) o nessuna spedizione trovata (per canale) | Verificare l'identificatore, il `channel_code` e il token usato |
| `403` | `Has no permission!` | Richiedere al supporto il permesso di consultazione dello status |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Account bloccato: contattare il supporto |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificare il token inviato in `Authorization` |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Attendere la registrazione del pagamento; se persiste, contattare il supporto |
| `422` | `O campo channel code deve conter um número inteiro.` (in `data.channel_code`) | Inviare un intero in `channel_code` oppure omettere il segmento |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar status da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Ripetere con backoff; se persiste, contattare il supporto |

## Note finali
- `data.carrier.timeline` è la vista più completa; i gruppi sono un riepilogo operativo per facilitare la
  lettura programmatica.
- Conserva il `tracking_number` per offrire al cliente finale il tracciamento sul sito del corriere.
- Per scaricare etichetta e invoice non appena la spedizione raggiunge `shipped`, vedi
  [Documenti della Spedizione](api-documentos.md).
- Per creare e annullare spedizioni, vedi [Spedizione (v1)](api-remessa-v1.md) e [Spedizione (v2)](api-remessa-v2.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
