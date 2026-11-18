# Pianificazione del Ritiro

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md) e [Spedizione (v1)](api-remessa-v1.md)

## Panoramica
- Richiede al corriere di **prelevare** il collo presso il tuo indirizzo, invece di portarlo tu a un punto
  di spedizione.
- Permette di consultare lo status della pianificazione e di annullarla con un motivo standardizzato.
- Funziona per le spedizioni create in **v1 e in v2**.

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

## Prerequisiti

Prima di pianificare, verifica che:

1. La spedizione sia già nello status **`shipped`** ([tracciamento](api-rastreio.md)) — ovvero sia già stata elaborata presso il
   corriere e i documenti esistano.
2. Il corriere del contratto scelto **disponga di un servizio di ritiro integrato**.
3. La spedizione **non abbia** un'altra pianificazione in corso.

## Flusso di utilizzo

```
1. GET  /v1/shipping/{shipping}/status   → confermare lo status "shipped"
2. POST /v1/schedule                     → creare la pianificazione
3. GET  /v1/schedule/{shipping}          → seguire fino allo status "success"
4. DELETE /v1/schedule/{shipping}/{reason} → annullare (se necessario)
```

## Convenzioni
- Date nel formato `Y-m-d` (es.: `2026-03-10`).
- Orari nel formato `H:i:s` (es.: `08:00:00`).
- L'identificatore `{shipping}` accetta l'**UUID della quotazione** o il **numero dell'AWB** (spedizioni degli
  ultimi 90 giorni). Diversamente dagli endpoint di spedizione, **la chiave della NF-e non è accettata qui**.

## Endpoint: creare una pianificazione

- **Metodo e rotta:** `POST /v1/schedule`
- **Quando usarlo:** dopo che la spedizione ha raggiunto `shipped`.

### Parametri

| Campo | Tipo | Obbligatorio | Min. | Max. | Formato | Descrizione |
|---|---|---|---|---|---|---|
| `shipping` | string | **sì** se `awb_number` non viene inviato | - | - | UUID | Codice della quotazione/spedizione |
| `awb_number` | string | **sì** se `shipping` non viene inviato | - | 50 | - | Numero dell'AWB (spedizioni degli ultimi 90 giorni) |
| `date` | string | **sì** | - | - | `Y-m-d` | Data del ritiro. Deve essere **successiva a oggi** |
| `initial_time` | string | **sì** | - | - | `H:i:s` | Inizio della finestra di ritiro |
| `final_time` | string | **sì** | - | - | `H:i:s` | Fine della finestra di ritiro |
| `instruction` | string | **sì** | 5 | 80 | - | Istruzione per l'autista (es.: dove ritirare, con chi parlare) |
| `address` | object | no | - | - | - | Indirizzo alternativo di ritiro (vedi tabella sotto) |

> Invia **`shipping` o `awb_number`**, non entrambi.

### Parametri — `address` (indirizzo alternativo di ritiro)

Invialo solo se il ritiro avviene a un indirizzo **diverso** da quello del mittente della spedizione.
**L'indirizzo deve essere nello stesso paese del mittente.**

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `street` | string | **sì** se `address` viene inviato | 5 | 70 | Indirizzo (via) |
| `city` | string | **sì** se `address` viene inviato | 5 | 35 | Città |
| `name` | string | **sì** se `address` viene inviato | 5 | 35 | Nome del contatto sul posto |
| `email` | string | **sì** se `address` viene inviato | - | 100 | E-mail del contatto |
| `phone` | string | **sì** se `address` viene inviato | 10 | 25 | Telefono del contatto |
| `postal_code` | string | condizionale | - | 12 | Codice postale/ZIP. **Obbligatorio per i ritiri in Brasile** |
| `state_code` | string | no | - | 10 | Sigla dello stato. Deve appartenere al paese del mittente |
| `number` | string | condizionale | - | 6 | Numero civico. **Obbligatorio per i ritiri in Brasile** |
| `district` | string | condizionale | - | 30 | Quartiere. **Obbligatorio per i ritiri in Brasile** |
| `complement` | string | no | - | 20 | Complemento |
| `phone_extension` | string | no | - | 5 | Interno |
| `federal_tax_id` | string | condizionale | - | 20 | CPF/CNPJ. **Obbligatorio e validato per i ritiri in Brasile** |

> Quando `address` viene omesso, la piattaforma risolve l'indirizzo automaticamente: prima la preimpostazione di
> ritiro del tuo account e, in sua assenza, l'indirizzo del mittente della spedizione.
>
> In alcuni tipi di account (persona fisica e persona giuridica mono-CD), l'indirizzo alternativo viene **ignorato** e il ritiro
> usa sempre l'indirizzo del mittente.

### Payload minimo

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Ritirare alla reception, parlare con Carlos"
}
```

### Payload massimo

```json
{
  "awb_number": "271249639538",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Ritirare nel capannone 2, cancello laterale, parlare con Carlos",
  "address": {
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Capannone 2",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

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
    "instruction": "Ritirare alla reception, parlare con Carlos"
  }'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> La risposta conferma la **ricezione**. La conferma del corriere è asincrona: consulta la
> pianificazione fino a quando lo status diventa `success` e compare il `confirmation_code`.

### Regole su data e finestra

| Regola | Dettaglio | Messaggio in caso di fallimento |
|---|---|---|
| Data futura | `date` deve essere **successiva alla data attuale** | `A data de agendamento não pode ser menor que a data Atual` |
| Nessun fine settimana | Sabato e domenica non sono accettati | `Coleta não pode ser realizada em um final de semana` |
| Nessuna festività nazionale | Per i ritiri in Brasile | `Coleta não pode ser realizada durante um feriado nacional` |
| Limite di 7 giorni lavorativi | La data non può superare i 7 giorni lavorativi a partire da oggi | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Festività nazionali brasiliane considerate: **01/01, 21/04, 01/05, 07/09, 12/10, 02/11, 15/11, 20/11 e 25/12**.

## Endpoint: consultare la pianificazione

- **Metodo e rotta:** `GET /v1/schedule/{shipping}`

| Parametro | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `shipping` | path | string | **sì** | UUID della quotazione o numero dell'AWB (ultimi 90 giorni) |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Ritirare alla reception, parlare con Carlos",
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
| `data.confirmation_code` | string | Numero di conferma del ritiro presso il corriere. Compilato quando lo status arriva a `success` |
| `data.date` | string | Data del ritiro (`Y-m-d`) |
| `data.inicial_time` | string | Inizio della finestra (`H:i:s`). ⚠️ **La chiave è `inicial_time`** nella risposta, mentre nella request il campo è `initial_time` |
| `data.final_time` | string | Fine della finestra (`H:i:s`) |
| `data.instruction` | string | Istruzione indicata alla creazione |
| `data.status.code` | string | Codice dello status della pianificazione (vedi tabella) |
| `data.status.description` | string | Descrizione tradotta in base all'header `lang` |
| `data.status.reason` | string \| null | Codice del motivo di annullamento (`001`–`008`). `null` finché non c'è un annullamento |

### Status della pianificazione

| `code` | Significato |
|---|---|
| `processing` | Pianificazione ricevuta, in attesa dell'invio al corriere |
| `success` | **Ritiro confermato dal corriere.** `confirmation_code` disponibile |
| `canceling` | Annullamento richiesto, in attesa di elaborazione |
| `terminating` | Annullamento in elaborazione presso il corriere |
| `canceled` | Annullamento completato |
| `error` | Errore nella pianificazione o nell'annullamento |

### Tipo di elaborazione

La piattaforma classifica la pianificazione automaticamente, senza alcun campo nel payload:

| Tipo | Quando | Comportamento |
|---|---|---|
| `automatico` | Ritiro per il giorno successivo (fino a 1 giorno) | Inviato al corriere immediatamente |
| `agendado` | Ritiro con più di 1 giorno di anticipo | Inviato al corriere 1 giorno prima del ritiro |

> Per questo motivo una pianificazione futura può restare in `processing` per giorni. È un comportamento previsto.

## Endpoint: annullare la pianificazione

- **Metodo e rotta:** `DELETE /v1/schedule/{shipping}/{reason}`
- **Quando usarlo:** quando il ritiro è già stato **confermato** (status `success`) e deve essere disdetto.

| Parametro | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `shipping` | path | string | **sì** | UUID della quotazione o numero dell'AWB (ultimi 90 giorni) |
| `reason` | path | string | **sì** | Codice del motivo (vedi tabella). Inviato al corriere |

### Motivi di annullamento

| Codice | Motivo |
|---|---|
| `001` | Package not ready (il collo non è pronto) |
| `002` | Rates too high (tariffe troppo alte) |
| `003` | Transit time too slow (tempo di transito troppo lento) |
| `004` | Take to service center or drop box (sarà portato a un punto di spedizione) |
| `005` | Commitment time not met (tempo concordato non rispettato) |
| `006` | Reason not given (motivo non indicato) |
| `007` | Other (altro) |
| `008` | Pickup modified (ritiro modificato) |

> Questi codici sono standardizzati dai corrieri e trasmessi a loro senza traduzione.
> Usa `006` o `007` quando nessuno di quelli specifici è applicabile.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Regole dell'annullamento

| Regola | Messaggio in caso di fallimento |
|---|---|
| Il `reason` deve essere compreso tra `001` e `008` | `Motivo do cancelamento não é compativel` |
| La pianificazione deve esistere | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| La spedizione deve essere in `shipped` | idem |
| La pianificazione deve essere in `success` (già confermata dal corriere) | idem |

> Dopo la richiesta, lo status passa a `canceling` con la descrizione
> `Solicitação de cancelamento recebida! Em breve será processada!`, poi `terminating` e, infine,
> `canceled` (o `error`).

## Regole di business consolidate

| Regola | Dettaglio |
|---|---|
| Status della spedizione | La spedizione deve essere in `shipped` |
| Pianificazione unica | Non è possibile avere due pianificazioni attive sulla stessa spedizione |
| Corriere | Il servizio deve supportare il ritiro integrato |
| Data | Futura, giorno lavorativo, senza festività nazionali (BR), fino a 7 giorni lavorativi |
| Indirizzo BR | Richiede `postal_code`, `number`, `district` e un `federal_tax_id` valido (CPF o CNPJ) |
| Paese dell'indirizzo | Lo `state_code` deve appartenere al paese del mittente della spedizione |
| Proprietà | È possibile pianificare/consultare/annullare solo le spedizioni del tuo stesso account |
| AWB | Risolve solo le spedizioni degli **ultimi 90 giorni** |

## Errori comuni

| Status | Messaggio | Come correggere |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Consultare la pianificazione esistente prima di crearne un'altra |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Attendere lo status `shipped` |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Portare il collo a un punto di spedizione |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Usare una data futura |
| `400` | `Coleta não pode ser realizada em um final de semana` | Scegliere un giorno lavorativo |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Scegliere un altro giorno lavorativo |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Ridurre l'anticipo |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Completare `number` e `district` (ritiri in Brasile) |
| `400` | `O atributo federal_tax_id não é válido` | Indicare un CPF o CNPJ valido |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Usare uno `state_code` del paese del mittente |
| `400` | `Motivo do cancelamento não é compativel` | Usare un codice compreso tra `001` e `008` |
| `403` | Permesso mancante | Richiedere al supporto il permesso di pianificazione |
| `404` | `Agendamento não foi encontrado para esta remessa` | Verificare l'identificatore o creare la pianificazione |
| `422` | Validazione | Correggere data/ora/istruzione secondo i formati |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- Consulta sempre la pianificazione prima di crearla o annullarla: questo evita richieste ridondanti e l'errore
  `já tem um agendamento em progresso`.
- Attenzione alla differenza di nomenclatura: **`initial_time` nella request** e **`inicial_time` nella risposta**.
- Le pianificazioni future restano in `processing` fino a 1 giorno prima del ritiro. Non è un errore.
