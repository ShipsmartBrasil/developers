# Primi Passi

- [Torna al README della lingua](README.md)

## Panoramica
Questa pagina raccoglie tutto ciò che è comune a **tutti** gli endpoint dell'API Pubblica di Shipsmart Global:
ambienti, autenticazione, lingua, formato della risposta, versionamento e gestione degli errori.

Leggi questa pagina prima di qualsiasi altra. Le altre pagine della guida presuppongono che tu conosca già
queste convenzioni e non le ripetono in dettaglio.

## Ambienti

| Ambiente | Base URL | Documentazione interattiva |
|---|---|---|
| Produzione | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Staging (omologazione) | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

- Usa lo **staging** per sviluppare e testare l'integrazione. Nessuna spedizione creata lì viene realmente spedita.
- Le nuove feature compaiono prima in staging. Se un campo descritto qui non esiste ancora in produzione,
  verificalo su `https://hmlapi.shipsmart.com.br/doc`.
- Accedere alla radice (`/`) reindirizza a `/doc`.

## Autenticazione

Ogni chiamata (tranne la documentazione) richiede un token di accesso personale nell'header `Authorization`,
secondo lo schema **Bearer**.

```
Authorization: Bearer IL_TUO_TOKEN
```

| Elemento | Valore |
|---|---|
| Tipo | Bearer token |
| Header | `Authorization` |
| Dove ottenerlo | Pannello di Shipsmart Global, nell'area delle impostazioni del tuo account |
| Scadenza | Il token non scade nel tempo; viene revocato se l'account viene bloccato o il token rigenerato |

### Errori di autenticazione

| Status | Risposta | Significato |
|---|---|---|
| `404` | `{"status":"warning","message":"..."}` | Token non trovato/non valido |
| `403` | `{"status":"warning","message":"..."}` | Account bloccato |
| `403` | `{"status":"warning","message":"Has no permission!"}` | Il token è valido, ma il tuo profilo non ha il permesso per l'azione |

> I permessi sono granulari per azione (quotare, spedire, annullare, elencare documenti, pianificare il ritiro ecc.).
> Se ricevi `Has no permission!`, richiedi al supporto l'abilitazione del permesso corrispondente.

## Lingua delle risposte

I messaggi di successo, avviso ed errore sono tradotti in base all'header `lang`.

| Header | Valori accettati | Predefinito |
|---|---|---|
| `lang` | `pt`, `en`, `es` | `pt` |

```
lang: en
```

> L'header è opzionale. Se non viene inviato (o viene inviato con un valore sconosciuto), l'API risponde in portoghese.

> **Sugli esempi di questa guida:** tutti i messaggi di risposta riprodotti nelle pagine seguenti
> usano la lingua predefinita dell'API (`pt`). Inviando `lang: en` o `lang: es`, il testo di `message` arriva
> tradotto, ma lo `status` e il codice HTTP sono sempre gli stessi.

## Convenzioni di richiesta

| Elemento | Valore |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Codifica | UTF-8 |
| Date (risposta) | `Y-m-d H:i:s` |
| Date (pianificazione) | `Y-m-d` per la data e `H:i:s` per l'ora |
| Valute | Codice ISO 4217 a 3 lettere (es.: `BRL`, `USD`, `EUR`) |
| Paesi | Codice ISO 3166-1 alpha-2 a 2 lettere (es.: `BR`, `US`, `PT`) |
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
puntando alla stessa risorsa su `/v1`, preservando la query string.

```
GET /quotation  →  308  →  GET /v1/quotation
```

> Per le nuove integrazioni, indica sempre la versione in modo esplicito. Non dipendere dal reindirizzamento.

### Compatibilità tra versioni
Una quotazione creata in v1 **può** generare solo una spedizione in v1, e una quotazione creata in v2 **può**
generare solo una spedizione in v2. Mischiare le versioni restituisce `422` con il messaggio
`Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.`

Vedi [Quotazione & Spedizione: v1 vs v2](api-comparativo-cotacao-remessa.md) per scegliere la versione.

## Formato standard delle risposte

Tutte le risposte seguono la stessa struttura:

```json
{
  "status": "success",
  "message": "Messaggio leggibile dagli esseri umani",
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
| `warning` | `400`, `403`, `404`, `406`, `422` | Una regola di business o una validazione ha impedito l'operazione. **Il payload deve essere corretto** |
| `error` | `500` | Guasto inatteso lato server. Riprova; se persiste, contatta il supporto |

## Errori di validazione (`422`)

Quando un campo non supera la validazione, la risposta include `data` con la mappa `campo → elenco di messaggi`.

```json
{
  "status": "warning",
  "message": "Validation Error",
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
> usa l'indice (`items.0.hscode`). Usa questa informazione per segnalare l'errore nel tuo form/integrazione.

## Tabella generale degli status HTTP

| Status | Quando si verifica | Cosa fare |
|---|---|---|
| `200` | Richiesta accettata/elaborata | Proseguire il flusso |
| `400` | Una regola di business ha bloccato l'operazione (es.: tipo di pagamento inesistente, quotazione scaduta, nessun corriere eleggibile) | Leggere `message` e correggere lo scenario |
| `403` | Permesso mancante o account bloccato | Richiedere l'abilitazione al supporto |
| `404` | Risorsa non trovata (token, spedizione, contratto non elencato nella quotazione) | Verificare l'identificatore indicato |
| `406` | Struttura del payload incompatibile con quella attesa | Rivedere i tipi e l'annidamento dei campi |
| `422` | Fallimento nella validazione dei campi | Correggere i campi elencati in `data` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Preimpostazioni dell'account (presets)

Alcune opzioni possono essere **preconfigurate sul tuo account** nel pannello. Quando sono configurate, vengono
applicate automaticamente nella quotazione e nella spedizione, e puoi omettere il campo nel payload.

| Preset | Influisce su | Nota |
|---|---|---|
| Canale di vendita (`channel_code`) | Quotazione | Usato quando non indichi il canale |
| Imposta (`tax`) | Quotazione | `sender` o `receiver` |
| Assicurazione (`insurance`) | Quotazione | - |
| Oggetto (`object`) e tipo (`type`) | Quotazione | - |
| Valute (`currency_quote`, `currency_payment`) | Quotazione | - |
| Sistema di misura (`measurement`) | Quotazione | `metric` o `imperial` |
| Consegna residenziale / non impilabile | Quotazione | - |
| Indirizzo del mittente | Quotazione e spedizione | Applicato come `address_code` |
| Nolo dichiarato (`declared_freight`) | Quotazione | Valore + valuta |
| Soggetto fatturante (`biller`) | Spedizione | Indirizzo mostrato sull'invoice |
| Raggruppamento delle scatole | Quotazione | `default`, `all_in_one` o `smartbox` |

**Precedenza:** ciò che invii nel payload vince sempre sul preset. Eccezioni:
- Se indichi `address_sender.country_code` nella quotazione, il preset dell'indirizzo del mittente viene scartato.
- Se indichi `address_sender.street` nella spedizione, il preset dell'indirizzo del mittente viene scartato.
- Se indichi `declared_freight`, il preset del nolo dichiarato viene scartato.

### Raggruppamento delle scatole
Il raggruppamento definisce come gli articoli vengono distribuiti nelle scatole quando usi `type: items`.
**Non** è un campo del payload: è una configurazione del tuo account.

| Modalità | Comportamento |
|---|---|
| `default` | Raggruppamento predefinito della piattaforma |
| `all_in_one` | Consolida tutto in una sola scatola |
| `smartbox` | Sistema gli articoli nelle scatole registrate considerando volume e dimensioni (con il 10% di margine) e rispettando `is_groupable` dell'articolo |

## Valute accettate

Stesso elenco per `currency_quote` (valuta della quotazione) e `currency_payment` (valuta del pagamento):

`USD`, `CAD`, `NZD`, `TWD`, `HKD`, `AUD`, `SGD`, `BRL`, `MXN`, `ARS`, `CLP`, `COP`, `UYU`, `BOB`,
`PEN`, `PYG`, `EUR`, `CNY`, `GBP`, `JPY`, `CHF`, `THB`, `DKK`, `NOK`, `SEK`, `AED`, `TRY`, `ILS`,
`ZAR`, `SAR`, `RUB`, `INR`, `PLN`

> `currency_payment` con pagamento tramite gateway ha una disponibilità che dipende dal provider:
> `BRL` è elaborato da un provider e le altre valute da un altro. Se la valuta scelta non è
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

1. **Conserva sempre il `quotation`** restituito dalla quotazione: è la chiave di tutto il resto del flusso
   (spedizione, status, documenti, pianificazione).
2. **Registra anche il tuo identificatore commerciale** in `channel_identifier_number`. Questo permette di consultare
   status, documenti e annullare usando il numero del tuo ordine, senza dipendere dal `quotation`.
3. **Non fare polling aggressivo.** Le consultazioni di status/documenti devono avere un intervallo di alcuni
   secondi e un backoff esponenziale.
4. **Tratta `warning` come errore di business** (non ripetere la stessa richiesta) ed `error` come transitorio
   (può essere ripetuto).
5. **Rispetta i limiti di dimensione dei campi** descritti in ogni pagina. I testi più lunghi vengono rifiutati
   con `422`, non troncati.
6. **Le quotazioni scadono.** Crea la spedizione entro la validità indicata in `valid_until` di ogni corriere.

## Prossimi passi

- [Quotazione](api-cotacao-v1.md) o [Quotazione V2](api-cotacao-v2.md)
- [Spedizione](api-remessa-v1.md) o [Spedizione V2](api-remessa-v2.md)
- [Quotazione & Spedizione: v1 vs v2](api-comparativo-cotacao-remessa.md)
