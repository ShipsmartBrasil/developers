# Pianificazione del Ritiro

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md) e [Spedizione (v1)](api-remessa-v1.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Chiede al corriere di **ritirare** il collo presso il tuo indirizzo, invece di portarlo tu a un punto
  di consegna.
- Consente di consultare l'avanzamento della pianificazione e di annullarla con un motivo standardizzato.
- Funziona per le spedizioni create in **v1 e in v2** (gli endpoint di pianificazione esistono solo in `/v1`).
- La richiesta è **asincrona**: l'API registra la richiesta e l'invio al corriere avviene in
  background. La conferma arriva in seguito, nella consultazione della pianificazione.
- Ogni spedizione accetta **una sola pianificazione** tramite l'API.

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

> `Content-Type` è necessario solo nel `POST`. Ogni operazione richiede un permesso specifico nel tuo profilo
> (creare, consultare e annullare la pianificazione). Senza il permesso, la risposta è `403` con `Has no permission!`.

## Flusso di utilizzo

```
1. GET    /v1/shipping/{shipping}/status    → confermare che lo status ATTUALE sia "shipped"
2. POST   /v1/schedule                      → registrare la pianificazione (status "processing")
3. GET    /v1/schedule/{shipping}           → seguire fino allo status "success"
4. DELETE /v1/schedule/{shipping}/{reason}  → annullare, se necessario (solo con status "success")
```

### Prima di pianificare, verifica

| # | Condizione | Come verificare |
|---|---|---|
| 1 | Lo status **attuale** della spedizione è **`shipped`** (elaborata presso il corriere, documenti disponibili). Quando la spedizione avanza (es.: in transito), la pianificazione non viene più accettata | [Tracciamento](api-rastreio.md) |
| 2 | Il corriere del contratto della spedizione **dispone del ritiro integrato** (vedi [Corrieri con ritiro integrato](#corrieri-con-ritiro-integrato)) | Contratto scelto nella spedizione |
| 3 | La spedizione **non ha ancora** alcuna pianificazione — nemmeno una annullata o con errore | `GET /v1/schedule/{shipping}` deve rispondere `404` |
| 4 | La spedizione è stata creata dallo **stesso utente** del token | - |
| 5 | Il tuo account **non** ha il ritiro automatico abilitato nelle preimpostazioni (vedi [Ritiro automatico tramite le preimpostazioni dell'account](#ritiro-automatico-tramite-le-preimpostazioni-dellaccount)) | Pannello dell'account |

## Concetti essenziali

### Convenzioni

| Elemento | Formato | Esempio |
|---|---|---|
| Data del ritiro | `AAAA-MM-GG` (esattamente questo formato) | `2026-03-10` |
| Orari della finestra | `HH:MM:SS` (con i secondi) | `08:00:00` |
| Data "attuale" | Considera il fuso orario della piattaforma (ora di Brasília) | - |
| Identificatore `{shipping}` | **UUID della quotazione** o **numero dell'AWB** (spedizioni create negli ultimi 90 giorni) | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` |

> A differenza degli endpoint di spedizione, **la chiave della NF-e non è accettata** come identificatore nella pianificazione.
> Un identificatore che contiene un trattino (`-`) viene sempre trattato come UUID; senza trattino, viene trattato come AWB.

### Corrieri con ritiro integrato

| Corriere | Pianificazione tramite API | Annullamento tramite API |
|---|---|---|
| DHL | sì | sì |
| FedEx | sì | sì |
| UPS | sì | sì |
| SkyPostal | sì | sì |
| CTBI Log | sì | **no** (la richiesta di annullamento termina in `error`; contatta il supporto) |
| GLS | sì | **no** (la richiesta di annullamento termina in `error`; contatta il supporto) |

> Elenco valido alla data di questa versione della guida; può essere ampliato. Per qualsiasi altro corriere,
> il `POST` risponde `400` con `Esta transportadora não possui serviço de coleta integrado`.

### Indirizzo del ritiro

L'indirizzo di ritiro viene definito in uno di questi modi. **Non mescolarli**: o invii `address`, o lasci
che sia la piattaforma a determinarlo.

| Forma | Come inviare | Indirizzo usato |
|---|---|---|
| **Forma A — automatico** | Ometti `address` (o invia `null` / `{}`) | 1º la preimpostazione di ritiro del tuo account per il paese del mittente (se abilitata e con indirizzo registrato); 2º l'indirizzo del mittente della spedizione |
| **Forma B — indirizzo alternativo** | Invia l'oggetto `address` completo | L'indirizzo indicato, che deve trovarsi **nello stesso paese del mittente** |

> Negli account di tipo **persona fisica** e **PJ mono-CD**, l'oggetto `address` viene **ignorato** e il ritiro usa
> sempre l'indirizzo del mittente della spedizione. Viene verificato solo il formato dei campi (`422`); le regole
> sull'indirizzo (paese, CAP, documento) non vengono applicate.

### Status della pianificazione

| `code` | Significato | Passo successivo |
|---|---|---|
| `processing` | Pianificazione registrata, in attesa di invio/conferma da parte del corriere | Attendere e consultare di nuovo |
| `success` | **Ritiro confermato dal corriere.** `confirmation_code` disponibile | Tenere pronto il collo nella finestra indicata |
| `canceling` | Annullamento richiesto, in attesa di invio al corriere | Attendere |
| `terminating` | Annullamento inviato al corriere, in attesa di conferma | Attendere |
| `canceled` | Annullamento completato | - |
| `error` | Errore nella pianificazione o nell'annullamento. Il motivo si trova in `status.description` | Leggere la descrizione e contattare il supporto |

```
POST                         DELETE
 │                             │
 ▼                             ▼
processing ──► success ──► canceling ──► terminating ──► canceled
 │                             │              │
 └──► error                    └──► error     └──► error
```

### Tipo di elaborazione

La piattaforma classifica automaticamente la pianificazione in base alla data indicata. Non esiste un campo nel payload.

| Tipo | Quando | Comportamento |
|---|---|---|
| Immediato | `date` = domani | Inviato al corriere nel successivo ciclo di elaborazione |
| Programmato | `date` dopodomani o più avanti | Inviato al corriere **1 giorno prima** della data del ritiro |

> Per questo motivo una pianificazione programmata resta in `processing` per alcuni giorni. È un comportamento previsto.
> Esempio: una richiesta fatta di venerdì per il lunedì successivo è **programmata** (inviata la domenica).

### Ritiro automatico tramite le preimpostazioni dell'account

Se il tuo account ha il **ritiro automatico** abilitato nelle preimpostazioni (configurate nel pannello), la
piattaforma crea autonomamente la pianificazione non appena vengono generati i documenti della spedizione, senza chiamate all'API:

| Elemento | Valore applicato |
|---|---|
| Data | Giorno lavorativo successivo (per un mittente in Brasile, salta anche una festività nazionale) |
| Finestra | Dalle `08:00:00` alle `18:00:00` |
| Istruzione | L'istruzione registrata nella preimpostazione |
| Indirizzo | Stessa regola della Forma A (preimpostazione di ritiro e, in sua mancanza, il mittente) |

> In questo caso, un `POST /v1/schedule` per la stessa spedizione risponde
> `400 Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso`.
> Usa solo il `GET /v1/schedule/{shipping}` per seguirla.

### Motivi di annullamento

| Codice | Motivo |
|---|---|
| `001` | Package not ready (il collo non è pronto) |
| `002` | Rates too high (tariffe troppo alte) |
| `003` | Transit time too slow (tempo di transito troppo lento) |
| `004` | Take to service center or drop box (sarà portato a un punto di consegna) |
| `005` | Commitment time not met (tempo concordato non rispettato) |
| `006` | Reason not given (motivo non indicato) |
| `007` | Other (altro) |
| `008` | Pickup modified (ritiro modificato) |

> Questi codici sono standardizzati dai corrieri e vengono trasmessi loro senza traduzione.
> Usa `006` o `007` quando nessuno dei motivi specifici si applica. Invia sempre le **3 cifre** (`001`, non `1`).

## Endpoint: creare una pianificazione

- **Metodo e rotta:** `POST /v1/schedule`
- **Quando usarlo:** con la spedizione nello status attuale `shipped`, per richiedere il ritiro del collo.

### Parametri

| Campo | Tipo | Obbligatorio | Min. | Max. | Formato / valori | Descrizione |
|---|---|---|---|---|---|---|
| `shipping` | string | **sì** se `awb_number` non viene inviato | - | - | UUID di una quotazione esistente | Codice della quotazione/spedizione |
| `awb_number` | string | **sì** se `shipping` non viene inviato | - | 50 | - | Numero dell'AWB (spedizioni create negli ultimi 90 giorni) |
| `date` | string | **sì** | - | - | `AAAA-MM-GG` | Data del ritiro. Vedi [Regole sulla data](#regole-sulla-data) |
| `initial_time` | string | **sì** | - | - | `HH:MM:SS` | Inizio della finestra di ritiro |
| `final_time` | string | **sì** | - | - | `HH:MM:SS` | Fine della finestra di ritiro |
| `instruction` | string | **sì** | 5 | 80 | - | Istruzione per l'autista (dove ritirare, con chi parlare) |
| `address` | object \| null | no | - | - | - | Indirizzo alternativo di ritiro (Forma B). Vedi la tabella qui sotto |

> Invia **`shipping` o `awb_number`**, non entrambi. Se vengono inviati entrambi, prevale `awb_number`.
>
> L'API **non** confronta `initial_time` con `final_time` né convalida l'orario lavorativo. Assicurati che la fine sia
> successiva all'inizio e che la finestra sia accettata dal corriere; in caso contrario, la pianificazione può
> terminare in `error`.

### Parametri — `address` (indirizzo alternativo di ritiro)

Invialo solo se il ritiro avviene a un indirizzo **diverso** da quello automatico.

**Attenzione: quando `address` viene inviato, tutte le 12 chiavi seguenti devono essere presenti nell'oggetto.**
Una chiave mancante fa fallire la richiesta con `500`. Quando non disponi del dato, usa `null` nei campi che
accettano il valore nullo e `""` (testo vuoto) in `complement` e `phone_extension`.

| Campo | Tipo | Accetta `null` | Min. | Max. | Regola per mittente in Brasile | Descrizione |
|---|---|---|---|---|---|---|
| `street` | string | no | 5 | 70 | - | Via |
| `number` | string | sì | - | 6 | **Obbligatorio** (non può essere vuoto) | Numero civico |
| `district` | string | sì | - | 30 | **Obbligatorio** (non può essere vuoto) | Quartiere |
| `complement` | string | no (usa `""`) | - | 20 | - | Complemento |
| `city` | string | no | 5 | 35 | - | Città |
| `state_code` | string | sì | - | 10 | - | Sigla dello stato. Deve appartenere al paese del mittente |
| `postal_code` | string | sì | - | 12 | - | CAP/ZIP. Se `state_code` arriva vuoto, lo stato viene ricavato da esso |
| `name` | string | no | 5 | 35 | - | Nome del referente sul posto |
| `email` | string | no | - | 100 | - | E-mail del referente (formato e-mail valido) |
| `phone` | string | no | 10 | 25 | - | Telefono del referente |
| `phone_extension` | string | no (usa `""`) | - | 5 | - | Interno |
| `federal_tax_id` | string | sì | - | 20 | **Obbligatorio: CPF o CNPJ valido** | Documento del responsabile del luogo |

Come vengono verificati paese e stato:

| Situazione | Comportamento |
|---|---|
| `state_code` indicato | Deve essere uno stato del paese del mittente (o il codice stesso del paese). Altrimenti: `Endereço enviado não pertence ao pais da remessa` |
| `state_code` vuoto e `postal_code` indicato (mittente BR) | Lo stato viene ottenuto dal CEP. CEP non trovato: `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| `state_code` vuoto e `postal_code` indicato (altri paesi) | Lo stato viene ottenuto dal codice postale. Non trovato: `Informações de endereço de coleta insuficientes ou inválidas` |
| `state_code` e `postal_code` vuoti | Non viene effettuata alcuna verifica. Consigliamo di inviare sempre entrambi per evitare il rifiuto da parte del corriere |

### Payload minimo

Forma A con `shipping` (UUID):

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Ritirare alla reception"
}
```

Forma A con `awb_number`:

```json
{
  "awb_number": "1234567890",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Ritirare alla reception"
}
```

### Payload massimo

Forma B (indirizzo alternativo, mittente in Brasile), con tutti i campi accettati:

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Ritirare nel capannone 2, cancello laterale, chiedere di Carlos",
  "address": {
    "street": "Via Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Capannone 2",
    "city": "Citta Esempio",
    "state_code": "SP",
    "postal_code": "01000000",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

> Per usare l'AWB, sostituisci `shipping` con `awb_number`. Il `federal_tax_id` dell'esempio è fittizio: nelle
> spedizioni dal Brasile, l'API richiede un CPF o CNPJ valido.

### Esempio di request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2026-03-10",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Ritirare alla reception"
  }'
```

### Esempio di response

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> La risposta conferma solo la **registrazione** della richiesta. La conferma del corriere è asincrona:
> consulta la pianificazione finché lo status non diventa `success` e non compare il `confirmation_code`.

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success` quando la pianificazione è stata registrata |
| `message` | string | Messaggio tradotto in base all'header `lang` |

> Questa risposta non contiene `data`. Per ottenere i dettagli, usa [consultare la pianificazione](#endpoint-consultare-la-pianificazione).

### Regole sulla data

Le regole vengono verificate in questo ordine; la prima che fallisce determina il messaggio (`400`).

| Regola | Dettaglio | Messaggio in caso di errore |
|---|---|---|
| Data futura | `date` deve essere **successiva a oggi** (oggi non è accettato) | `A data de agendamento não pode ser menor que a data Atual` |
| Niente fine settimana | Sabato e domenica non sono accettati, per qualsiasi paese | `Coleta não pode ser realizada em um final de semana` |
| Niente festività nazionali | Solo quando il mittente si trova in Brasile | `Coleta não pode ser realizada durante um feriado nacional` |
| Limite di 7 giorni lavorativi | La data non può superare il 7º giorno lavorativo dopo oggi (per un mittente in Brasile, le festività nazionali non contano come giorni lavorativi) | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Festività nazionali brasiliane considerate (date fisse): **01/01, 21/04, 01/05, 07/09, 12/10, 02/11,
15/11, 20/11 e 25/12**. Le festività mobili (es.: Carnevale, Venerdì Santo, Corpus Domini) e le festività
statali/municipali **non** vengono bloccate dall'API — evita queste date per conto tuo.

### Ordine delle validazioni

| # | Verifica | Status | Messaggio |
|---|---|---|---|
| 1 | Formato dei campi | `422` | Messaggi per campo in `data` |
| 2 | AWB trovato (se è stato inviato `awb_number`) | `400` | `Agendamento não foi encontrado para esta remessa` |
| 3 | Data successiva a oggi | `400` | `A data de agendamento não pode ser menor que a data Atual` |
| 4 | Spedizione senza pianificazione precedente | `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` |
| 5 | Spedizione esistente e con status attuale `shipped` | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 6 | Corriere con ritiro integrato | `400` | `Esta transportadora não possui serviço de coleta integrado` |
| 7 | Regole sulla data (fine settimana, festività, 7 giorni lavorativi) | `400` | Vedi [Regole sulla data](#regole-sulla-data) |
| 8 | Spedizione creata dall'utente del token | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 9 | Indirizzo alternativo (se inviato) | `400` | Vedi [Errori comuni](#errori-comuni) |

## Endpoint: consultare la pianificazione

- **Metodo e rotta:** `GET /v1/schedule/{shipping}`
- **Quando usarlo:** per seguire la pianificazione dopo il `POST` e prima di annullarla.

### Parametri

| Parametro | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `shipping` | path | string | **sì** | UUID della quotazione o numero dell'AWB (spedizioni degli ultimi 90 giorni) |

### Payload minimo

Senza corpo. Solo l'identificatore nel path:

```
GET /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944
```

### Payload massimo

Non ci sono altri parametri. L'alternativa è usare l'AWB al posto dell'UUID:

```
GET /v1/schedule/1234567890
```

### Esempio di request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response

`200 OK`

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Ritirare alla reception",
    "status": {
      "code": "processing",
      "description": "Agendamento da Remessa Recebida. Em breve será processada!",
      "reason": null
    }
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.confirmation_code` | string \| null | Numero di conferma del ritiro presso il corriere. È definitivo solo quando `status.code` è `success`; prima può arrivare `null` o un valore provvisorio (es.: `000000`) |
| `data.date` | string | Data del ritiro (`AAAA-MM-GG`) |
| `data.inicial_time` | string | Inizio della finestra (`HH:MM:SS`). ⚠️ **La chiave è `inicial_time`** nella risposta, mentre nella request il campo è `initial_time` |
| `data.final_time` | string | Fine della finestra (`HH:MM:SS`) |
| `data.instruction` | string | Istruzione indicata nella creazione (o quella della preimpostazione, nel ritiro automatico) |
| `data.status.code` | string | Codice dello status della pianificazione (vedi [Status della pianificazione](#status-della-pianificazione)). Usa questo campo nella tua logica |
| `data.status.description` | string | Testo informativo dell'ultimo evento. **Non viene tradotto nella consultazione**: può arrivare nella lingua in cui è stata fatta la richiesta, in inglese (es.: `Pickup Scheduled Successfully!`, `Pickup Cancelled Successfully!`) oppure con il messaggio restituito dal corriere, in caso di `error` |
| `data.status.reason` | string \| null | Codice del motivo di annullamento (`001`–`008`). `null` finché non c'è una richiesta di annullamento |

## Endpoint: annullare la pianificazione

- **Metodo e rotta:** `DELETE /v1/schedule/{shipping}/{reason}`
- **Quando usarlo:** quando il ritiro è già stato **confermato** (status `success`) e deve essere disdetto.

### Parametri

| Parametro | Posizione | Tipo | Obbligatorio | Valori | Descrizione |
|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | UUID della quotazione o numero dell'AWB (spedizioni degli ultimi 90 giorni) |
| `reason` | path | string | **sì** | da `001` a `008` | Codice del motivo (vedi [Motivi di annullamento](#motivi-di-annullamento)). Trasmesso al corriere |

### Payload minimo

Senza corpo. Identificatore e motivo nel path:

```
DELETE /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/006
```

### Payload massimo

Non ci sono altri parametri. L'alternativa è usare l'AWB al posto dell'UUID:

```
DELETE /v1/schedule/1234567890/001
```

### Esempio di request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success` quando la richiesta di annullamento è stata registrata |
| `message` | string | Messaggio tradotto in base all'header `lang` |

> Dopo la richiesta, lo status passa a `canceling` con la descrizione
> `Solicitação de cancelamento recebida! Em breve será processada!`, poi a `terminating` e, infine,
> a `canceled` (o `error`). Consulta la pianificazione per seguirla.

### Regole dell'annullamento

| Regola | Status | Messaggio in caso di errore |
|---|---|---|
| Il `reason` deve essere uno dei codici da `001` a `008` | `422` | `O campo reason não contém um valor válido.` (in `data.reason`) |
| L'AWB deve essere trovato (se usato nel path) | `400` | `Agendamento não foi encontrado para esta remessa` |
| La pianificazione deve esistere | `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| La spedizione deve restare con status attuale `shipped` | `400` | idem |
| La pianificazione deve essere in `success` (già confermata). In `processing`, `canceling`, `terminating`, `canceled` o `error`, l'annullamento viene rifiutato | `400` | idem |

## Regole di business

| Regola | Dettaglio |
|---|---|
| Status della spedizione | Lo status **attuale** deve essere `shipped`, sia per pianificare sia per annullare |
| Pianificazione unica | Ogni spedizione accetta una sola pianificazione tramite l'API. Anche una pianificazione annullata o con errore **impedisce** un nuovo `POST`; per ripianificare, contatta il supporto |
| Corriere | Il servizio della spedizione deve avere il ritiro integrato. Annullamento integrato solo per DHL, FedEx, UPS e SkyPostal |
| Data | Successiva a oggi, giorno lavorativo, senza festività nazionali fisse (mittente BR), entro 7 giorni lavorativi |
| Finestra | `initial_time` e `final_time` nel formato `HH:MM:SS`; non c'è validazione dell'ordine né dell'orario lavorativo |
| Indirizzo alternativo | Tutte le 12 chiavi presenti; in Brasile richiede `number`, `district` e un `federal_tax_id` valido (CPF o CNPJ) |
| Paese dell'indirizzo | Sempre il paese del mittente della spedizione. Il `state_code` deve appartenere a tale paese |
| Tipo di account | Persona fisica e PJ mono-CD: `address` ignorato; ritiro all'indirizzo del mittente |
| Ritiro automatico | Con la preimpostazione di ritiro automatico attiva, la pianificazione viene creata dalla piattaforma e il `POST` manuale viene rifiutato |
| Proprietà | Pianifica e consulta solo spedizioni create dall'utente del token |
| AWB | Risolve solo spedizioni create negli **ultimi 90 giorni** (la più recente con quel numero) |
| Elaborazione | Immediata per il ritiro domani; programmata (inviata 1 giorno prima) per le date successive |

## Errori comuni

| Status | Significato/Messaggio | Come correggere |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Esiste già una pianificazione (manuale, automatica, annullata o con errore). Consultala con `GET`; per ripianificare, contatta il supporto |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Verificare che lo status attuale sia `shipped` e che la spedizione appartenga all'utente del token |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Portare il collo a un punto di consegna |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Usare una data a partire da domani |
| `400` | `Coleta não pode ser realizada em um final de semana` | Scegliere un giorno lavorativo |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Scegliere un altro giorno lavorativo |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Ridurre l'anticipo |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Mittente BR: compilare `number` e `district`. Altri paesi: verificare `postal_code` o inviare `state_code` |
| `400` | `O atributo federal_tax_id não é válido` | Mittente BR: indicare un CPF o CNPJ valido |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Usare un `state_code` del paese del mittente |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Verificare il `postal_code` o indicare lo `state_code` |
| `400` | `Agendamento não foi encontrado para esta remessa` | Nel `POST`/`DELETE` con AWB: verificare il numero, l'età della spedizione (90 giorni) e che appartenga al tuo account |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | Annulla solo pianificazioni in `success` di spedizioni in `shipped` |
| `403` | `Has no permission!` | Richiedere al supporto il permesso per l'operazione (creare, consultare o annullare la pianificazione) |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contattare il supporto |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificare l'header `Authorization: Bearer ...` |
| `404` | `Agendamento não foi encontrado para esta remessa` | Nel `GET` con UUID: la spedizione non ha una pianificazione o non appartiene al tuo account |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + mappa dei campi in `data` | Correggere i campi elencati (es.: `O campo data é obrigatório.`, `A data informada para o campo :attribute não respeita o formato :format.`, `O valor selecionado para o campo shipping é inválido.`, `O campo reason não contém um valor válido.`) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | `POST`: verificare che siano state inviate tutte le chiavi di `address`. `GET` con AWB: l'AWB non è stato trovato (inesistente, con più di 90 giorni o di un altro account). Negli altri casi, ripetere con backoff e contattare il supporto |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Errore imprevisto nel `DELETE`. Ripetere con backoff; se persiste, contattare il supporto |

> Attenzione: nella **consultazione per AWB** (`GET /v1/schedule/{awb}`), un AWB non trovato risponde `500`, e non
> `404`. Preferisci consultare tramite l'UUID della quotazione, che risponde `404` quando non c'è una pianificazione.

## Note finali
- Consulta sempre la pianificazione prima di crearla o annullarla: questo evita richieste ridondanti e l'errore
  `já tem um agendamento em progresso`.
- Attenzione alla differenza di nomenclatura: **`initial_time` nella request** e **`inicial_time` nella risposta**.
- Le pianificazioni programmate restano in `processing` fino a 1 giorno prima del ritiro. Non è un errore. Se lo status
  non cambia dopo la data del ritiro, contatta il supporto.
- Usa `status.code` (e non `status.description`) per la logica della tua integrazione.
- Status della spedizione: [Tracciamento](api-rastreio.md). Documenti per il ritiro: [Documenti](api-documentos.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
