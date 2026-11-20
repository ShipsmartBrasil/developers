# Primi Passi

- [Torna al README della lingua](README.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
Questa pagina raccoglie tutto ciò che è comune a **tutti** gli endpoint dell'API Pubblica di Shipsmart Global:
ambienti, autenticazione, lingua, formato della risposta, versionamento e gestione degli errori.

Leggi questa pagina prima di qualsiasi altra. Le altre pagine della guida presuppongono che tu conosca già
queste convenzioni e non le ripetono in dettaglio.

## Ambienti

| Ambiente | URL di base | Documentazione interattiva |
|---|---|---|
| Produzione | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Staging (homologazione) | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

- Usa lo **staging** per sviluppare e testare l'integrazione. Nessuna spedizione creata lì viene spedita realmente.
- Le nuove feature compaiono prima in staging. Se un campo descritto qui non esiste ancora in produzione,
  verificalo su `https://hmlapi.shipsmart.com.br/doc`.
- Accedere alla radice (`/`) reindirizza a `/doc`.

## Autenticazione

Ogni chiamata (eccetto la documentazione) richiede un token di accesso personale nell'header `Authorization`,
con lo schema **Bearer**.

```
Authorization: Bearer IL_TUO_TOKEN
```

| Elemento | Valore |
|---|---|
| Tipo | Bearer token |
| Header | `Authorization` |
| Dove ottenerlo | Pannello di Shipsmart Global, nell'area delle impostazioni del tuo account |
| Scadenza | Il token non scade nel tempo; viene revocato se l'account viene bloccato o se il token viene rigenerato |

### Errori di autenticazione

| Status | Risposta | Significato |
|---|---|---|
| `404` | `{"status":"warning","message":"Nenhum usuário encontrado para o token especificado!"}` | Token assente, non trovato o non valido |
| `403` | `{"status":"warning","message":"Usuário especificado encontra-se bloqueado em nosso sistema!"}` | Account bloccato |
| `403` | `{"status":"warning","message":"Has no permission!"}` | Il token è valido, ma il tuo profilo non ha il permesso per l'azione |

> I permessi sono granulari per azione (quotare, spedire, annullare, elencare documenti, pianificare il ritiro ecc.).
> Se ricevi `Has no permission!`, richiedi al supporto l'abilitazione del permesso corrispondente.

## Lingua delle risposte

I messaggi di successo, avviso ed errore vengono tradotti in base all'header `lang`.

| Header | Valori accettati | Predefinito |
|---|---|---|
| `lang` | `pt`, `en`, `es` | `pt` |

```
lang: en
```

> L'header è opzionale. Se non viene inviato, l'API risponde in portoghese. Se viene inviato con un valore
> non supportato (es.: `fr`), i messaggi arrivano in inglese.

> **Sugli esempi di questa guida:** tutti i messaggi di risposta riprodotti nelle pagine seguenti
> usano la lingua predefinita dell'API (`pt`). Inviando `lang: en` o `lang: es`, il testo di `message` arriva
> tradotto, ma lo `status` e il codice HTTP sono sempre gli stessi.

## Convenzioni della richiesta

| Elemento | Valore |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Codifica | UTF-8 |
| Date (risposta) | `AAAA-MM-GG HH:MM:SS` |
| Date (pianificazione) | `AAAA-MM-GG` per la data e `HH:MM:SS` per l'ora |
| Valute | Codice ISO 4217 di 3 lettere (es.: `BRL`, `USD`, `EUR`) |
| Paesi | Codice ISO 3166-1 alpha-2 di 2 lettere (es.: `BR`, `US`, `PT`) |
| Separatore decimale | Punto (`.`) — mai la virgola |
| Pesi | Chilogrammi quando `measurement` è `metric`; libbre quando è `imperial` |
| Dimensioni | Centimetri quando `measurement` è `metric`; pollici quando è `imperial` |
| Volume | m³ quando `measurement` è `metric`; ft³ quando è `imperial` |

## Versionamento

La versione fa parte del path: `/v1/...` e `/v2/...`.

- `POST /v1/quotation` → quotazione nella prima versione
- `POST /v2/quotation` → quotazione nella seconda versione

### Chiamate senza prefisso di versione
Se chiami una risorsa **senza** il prefisso di versione, l'API risponde `308 Permanent Redirect`
puntando alla stessa risorsa in `/v1`, preservando la query string.

```
GET /quotation  →  308  →  GET /v1/quotation
```

> Per le nuove integrazioni, indica sempre la versione in modo esplicito. Non fare affidamento sul reindirizzamento.

### Compatibilità tra versioni
Una quotazione creata nella v1 può generare una spedizione **solo** nella v1, e una quotazione creata nella v2 può generare
una spedizione **solo** nella v2. Mescolare le versioni restituisce `422` con il messaggio (in `data.quotation`)
`Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.`

Consulta [Quotazione & Spedizione: v1 vs v2](api-comparativo-cotacao-remessa.md) per scegliere la versione.

## Formato standard delle risposte

Tutte le risposte seguono lo stesso envelope:

```json
{
  "status": "success",
  "message": "Messaggio leggibile per gli esseri umani",
  "data": {}
}
```

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success`, `warning` o `error` |
| `message` | string | Messaggio già tradotto in base all'header `lang` |
| `data` | object/array | Contenuto della risposta. Assente nelle risposte che confermano solo la ricezione |

### Relazione tra `status` e HTTP

| `status` | HTTP tipico | Interpretazione |
|---|---|---|
| `success` | `200` | L'operazione è stata accettata/completata |
| `warning` | `400`, `401`, `403`, `404`, `406`, `422` | Una regola di business o una validazione ha impedito l'operazione. **Il payload deve essere corretto** |
| `error` | `500` (e alcuni rifiuti di peso/corriere con `400`/`404`) | Errore inatteso o rifiuto del corriere. Leggi `message`; gli errori `500` possono essere ripetuti |

## Errori di validazione (`422`)

Quando un campo non supera la validazione, la risposta contiene `data` con la mappa `campo → elenco di messaggi`.
Il `message` di primo livello è **generico** (non descrive l'errore): usa sempre il contenuto di `data`.

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "object": ["O campo object é obrigatório."],
    "address_sender.country_code": [
      "O campo address sender.country code é obrigatório quando address sender.address code não está presente."
    ],
    "boxes": ["O campo boxes é obrigatório a menos que type esteja presente em items."]
  }
}
```

> Nei campi annidati, la chiave usa il punto (`address_sender.country_code`) e, negli array,
> usa l'indice (`items.0.hscode`). Usalo per indicare l'errore nel tuo modulo/integrazione.

## Tabella generale degli status HTTP

| Status | Quando si verifica | Cosa fare |
|---|---|---|
| `200` | Richiesta accettata/elaborata | Proseguire con il flusso |
| `308` | Chiamata senza prefisso di versione | Usare `/v1/...` o `/v2/...` |
| `400` | Una regola di business ha bloccato l'operazione (es.: documento fiscale non valido, scatola senza articolo, CAP non valido) | Leggere `message` e correggere lo scenario |
| `401` | Tipo di pagamento non abilitato per l'account | Abilitare l'opzione di pagamento o usare un altro tipo |
| `403` | Senza permesso, account bloccato, quotazione semplice o scaduta nella spedizione | Richiedere l'abilitazione al supporto / rifare la quotazione |
| `404` | Risorsa non trovata (token, quotazione di un altro account, spedizione, contratto non elencato nella quotazione, indirizzo) | Verificare l'identificatore indicato |
| `406` | Conflitto di business (quotazione già usata, indirizzo registrato incompatibile, carico che richiede assistenza personalizzata) | Leggere `message` e adeguare il flusso |
| `422` | Errore di validazione dei campi, o regola di business sui dati (es.: HS Code senza tabella di imposta, account solo DDP) | Correggere i campi elencati in `data` / leggere `message` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Preimpostazioni dell'account (presets)

Alcune opzioni possono essere **preconfigurate nel tuo account** dal pannello. Vengono applicate dall'API dopo la
validazione del payload, con regole diverse per campo:

| Preimpostazione | Riguarda | Come viene applicata |
|---|---|---|
| Imposta (`tax`) | Quotazione, Manifesto, Bozza | **Sostituisce** il `tax` inviato. I paesi di destinazione configurati come eccezione ricevono l'opposto del valore predefinito. Se l'account è **vincolato a DDP**, l'imposta è sempre `sender` |
| Assicurazione automatica | Quotazione, Manifesto | Attiva l'assicurazione quando il valore dichiarato (convertito in USD) raggiunge il minimo configurato, anche con `insurance: false` |
| Canale di vendita (`channel_code`) | Quotazione, Manifesto, Bozza | Usato quando il campo non viene inviato |
| Sistema di misura (`measurement`) | Quotazione, Manifesto, Bozza | Usato quando il campo non viene inviato; in sua mancanza, `metric` |
| Consegna residenziale / non impilabile | Quotazione, Bozza | Usati quando i campi non vengono inviati |
| Nolo dichiarato (`declared_freight`) | Quotazione, Manifesto, Bozza | Usato quando il campo non viene inviato. Richiede il permesso di nolo dichiarato |
| Raggruppamento delle scatole | Quotazione v2 (`type: items`) | Definisce come vengono composte le scatole (vedi sotto). Non è un campo del payload |
| Centro di costo (`cost_center_code`) | Spedizione v1/v2, Manifesto, Bozza | Usato quando il campo non viene inviato **o viene inviato vuoto/nullo** |
| Finalità della spedizione (`type` della spedizione) | Spedizione v1/v2, Bozza | Usata quando il campo non viene inviato |
| Soggetto fatturante (`biller`) | Spedizione v1/v2, Manifesto, Bozza | Usato quando il campo non viene inviato (o arriva vuoto) e l'account è configurato per mostrare il soggetto fatturante |
| Notifica di addebito (`payment_notify_shipping_billing`) | Spedizione v1/v2 | Usata quando il campo non viene inviato |
| Ritiro automatico | Spedizione | Crea automaticamente la pianificazione del ritiro (vedi [Pianificazione del Ritiro](api-agendamento.md)) |

**Precedenza:** eccetto per l'**imposta** (che segue sempre la preimpostazione, quando configurata), ciò che invii
nel payload prevale sulla preimpostazione. I campi obbligatori del payload (`object`, `type`, `insurance`, valute,
`address_sender`) devono essere sempre inviati — la preimpostazione non li rende facoltativi.

> L'indirizzo del mittente preconfigurato nel pannello **non** sostituisce l'`address_sender` dell'API.
> Per riutilizzare un indirizzo registrato, invia `address_sender.address_code`.

### Raggruppamento delle scatole
Il raggruppamento definisce come gli articoli vengono distribuiti nelle scatole quando usi `type: items`
(quotazione per articoli, **esclusiva della v2**).
**Non** è un campo del payload: è un'impostazione del tuo account.

| Modalità | Comportamento |
|---|---|
| `default` | Ogni articolo usa la scatola collegata alla sua registrazione (`box_code`/`box_identifier`); gli articoli raggruppabili con la stessa scatola vengono riuniti e gli articoli non raggruppabili ricevono una scatola per unità |
| `all_in_one` | Consolida gli articoli raggruppabili in un'unica scatola virtuale con il volume totale (a partire dalle dimensioni registrate); gli articoli non raggruppabili ricevono una scatola per unità |
| `smartbox` | Inserisce gli articoli nelle scatole **inviate in `boxes[]`** (obbligatorio), considerando volume e dimensioni con un margine del 10% e rispettando `is_groupable`/`is_foldable` dell'articolo |

Esempi di ciascuna modalità in [Quotazione di Articoli](api-cotacao-v2.md#quotazione-di-articoli).

## Registrazioni effettuate nel pannello

Alcuni codici usati nei payload provengono da registrazioni gestite **nel pannello di Shipsmart Global** (non esiste un endpoint
pubblico per crearle). Devono esistere, essere attive e appartenere al tuo account (o agli account autorizzati):

| Registrazione | Campo nel payload | Dove viene usata |
|---|---|---|
| Indirizzo | `address_sender.address_code`, `address_receiver.address_code`, `manufacturer.address_code`, `biller.address_code` | Quotazione, Spedizione, Manifesto, Bozza |
| Canale di vendita | `channel_code` | Quotazione, Manifesto, Bozza, consultazioni e annullamento per canale |
| Centro di costo | `cost_center_code` | Spedizione (creazione e aggiornamento), Manifesto, Bozza |
| Kit | `boxes[].kit` | Quotazione, Manifesto, Bozza |

Anche scatole e articoli possono essere registrati tramite l'API: vedi [Scatola](api-caixa-v1.md) e [Articoli](api-itens-v1.md).

## Valute accettate

Stesso elenco per `currency_quote` (valuta della quotazione) e `currency_payment` (valuta del pagamento):

`USD`, `CAD`, `NZD`, `TWD`, `HKD`, `AUD`, `SGD`, `BRL`, `MXN`, `ARS`, `CLP`, `COP`, `UYU`, `BOB`,
`PEN`, `PYG`, `EUR`, `CNY`, `GBP`, `JPY`, `CHF`, `THB`, `DKK`, `NOK`, `SEK`, `AED`, `TRY`, `ILS`,
`ZAR`, `SAR`, `RUB`, `INR`, `PLN`

> La disponibilità di `currency_payment` con pagamento tramite gateway dipende dal provider:
> `BRL` viene elaborato da un provider e le altre valute da un altro. Se la valuta scelta non è
> abilitata per il tuo account, la spedizione restituisce `warning` al momento del pagamento.

## Prima chiamata (test di connettività)

L'endpoint più semplice per validare token e connettività:

```bash
curl --request GET \
  --url https://hmlapi.shipsmart.com.br/v1/platform/version \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Accept: application/json'
```

Risposta:

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "name": "Shipsmart Global API",
    "env": "staging",
    "version": "1.0.0"
  }
}
```

## Buone pratiche di integrazione

1. **Conserva sempre il `quotation`** restituito nella quotazione: è la chiave di tutto il resto del flusso
   (spedizione, status, documenti, pianificazione).
2. **Registra anche il tuo identificatore commerciale** in `channel_identifier_number`. Questo permette di consultare
   status, documenti e annullare usando il numero del tuo ordine, senza dipendere dal `quotation`.
3. **Non fare mai polling aggressivo.** Le consultazioni di status/documenti devono avere un intervallo di alcuni
   secondi e un backoff esponenziale.
4. **Tratta `warning` come errore di business** (non ripetere la stessa richiesta) e `error` come transitorio
   (puoi ripetere).
5. **Rispetta i limiti di lunghezza dei campi** descritti in ogni pagina. I testi più lunghi vengono rifiutati
   con `422`, non troncati.
6. **Le quotazioni scadono.** Crea la spedizione entro la validità indicata in `valid_until` di ciascun corriere.

## Prossimi passi

- [Quotazione](api-cotacao-v1.md) o [Quotazione V2](api-cotacao-v2.md)
- [Spedizione](api-remessa-v1.md) o [Spedizione V2](api-remessa-v2.md)
- [Quotazione & Spedizione: v1 vs v2](api-comparativo-cotacao-remessa.md)
- [Codici di Errore](api-codigos-de-erro.md)
