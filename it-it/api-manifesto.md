# Manifesto

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Esegue **quotazione + spedizione + invio al corriere + generazione dell'etichetta in un'unica chiamata sincrona**.
- La risposta contiene già l'identificatore della spedizione, il numero di tracciamento e l'etichetta (AWB) in PDF e/o ZPL.
- È il flusso indicato per le operazioni che necessitano dell'etichetta **immediatamente**, senza attendere
  un'elaborazione asincrona.
- Considera solo i contratti di corrieri con spedizione sincrona supportata (attualmente **FedEx, UPS e DHL**).
- Accetta solo il **pagamento fatturato** (`payment.type: billed`), e l'account deve avere questa forma di pagamento abilitata.

> ⚠️ **Il manifesto è sincrono e irreversibile.** Poiché quota e spedisce nella stessa richiesta, non esiste
> una fase intermedia di revisione. Convalida bene il payload prima di inviarlo.

## URL di base
- Produzione: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Documentazione interattiva: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticazione
```
Authorization: Bearer IL_TUO_TOKEN
Content-Type: application/json
Accept: application/json
lang: pt
```

> L'header `lang` accetta `pt`, `en` e `es` (predefinito `pt`). Il profilo del token deve avere il permesso di
> **creare manifesti**; senza di esso la risposta è `403` con `Has no permission!`.

## Flusso di utilizzo
1. Componi il payload completo (indirizzi, scatole, articoli, pagamento). Consulta la sezione
   [Indirizzo manuale: invia tutte le chiavi](#indirizzo-manuale-invia-tutte-le-chiavi).
2. `POST /v1/manifest?format=pdf` (o `zpl`, oppure senza `format` per riceverli entrambi).
3. Stampa l'etichetta ricevuta in `data.docs` (usa il `base64` o scaricala dalla `url`).
4. Conserva il `data.shipping` e usalo in [tracciamento](api-rastreio.md), [documenti](api-documentos.md) e
   [pianificazione del ritiro](api-agendamento.md).
5. Qualche istante dopo, consulta [Documenti della Spedizione](api-documentos.md) per ottenere i documenti
   complementari (invoice, packing list ecc.), generati in background.

## Concetti essenziali

### Manifesto vs. quotazione + spedizione

| Tema | Quotazione + Spedizione | Manifesto |
|---|---|---|
| Chiamate | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Scelta del corriere | La scegli tu dall'elenco restituito | La sceglie la piattaforma (o la indichi tu con `carrier_code`/`slug`) |
| Corrieri | Tutti i contratti dell'account | **Solo FedEx, UPS e DHL** |
| Etichetta (AWB) | Consultazione successiva in `/docs` | **Arriva nella risposta** |
| Pagamento | `billed`, `gateway`, `afterward` | **Solo `billed`** |
| Tipo di risposta | Asincrona | **Sincrona** |
| Revisione prima della spedizione | Sì | No |
| Campi obbligatori | Meno (divisi in 2 fasi) | **Di più** (tutto in una volta) |

Usa il manifesto quando: l'operatività è ad alto volume, il corriere è già definito da una regola di
business e devi stampare l'etichetta all'istante.

### Come viene scelto il corriere

| Cosa invii | Comportamento |
|---|---|
| Né `slug` né `carrier_code` | La piattaforma sceglie in base alla preferenza configurata nell'account: **più veloce** (tempo minore, predefinito) o **più economico** (valore minore nella valuta di pagamento) |
| `carrier_code` | È una **preferenza**. Se il contratto compare nel risultato della quotazione, viene usato. Se non compare: contratto inesistente o di un corriere compatibile con il manifesto → la piattaforma ricade sulla preferenza dell'account; contratto esistente di un corriere **non** compatibile con il manifesto → `404` |
| `slug` | **Fissa** il contratto: la quotazione valuta solo quello e lo `slug` **sovrascrive** qualsiasi `carrier_code` inviato. Se il contratto non è tra i contratti idonei del tuo account per quell'invio, la quotazione torna a valutarli tutti e la scelta segue la regola del `carrier_code` |

> Non mescolare `slug` e `carrier_code`: usa l'uno o l'altro. Se li invii entrambi, prevale lo `slug`.
> Quando l'account ha un contratto preferito preimpostato e non invii `carrier_code`, la preimpostazione viene usata come `carrier_code`.

### Preimpostazioni dell'account nel manifesto

Il manifesto applica le [preimpostazioni dell'account](api-primeiros-passos.md#preimpostazioni-dellaccount-presets)
**dopo** la validazione dell'input. Per questo motivo, tutti i campi obbligatori devono essere nel payload,
anche se l'account ha una preimpostazione per essi.

| Preimpostazione | Quando viene applicata nel manifesto |
|---|---|
| Responsabile dell'imposta (`tax`) | **Ogni volta che è configurata: sostituisce il `tax` inviato** (vedi la tabella qui sotto) |
| Centro di costo (`cost_center_code`) | Quando il payload non invia il campo o lo invia vuoto (`null`, `""` o `0`) |
| Canale di vendita (`channel_code`) | Quando la chiave `channel_code` non viene inviata. Inviare `"channel_code": null` disattiva la preimpostazione |
| Contratto preferito (`carrier_code`) | Quando la chiave `carrier_code` non viene inviata |
| Nolo dichiarato (`declared_freight`) | Quando la chiave `declared_freight` non viene inviata. Inviare `declared_freight` (anche `null` o `{}`) scarta la preimpostazione |
| Soggetto fatturante (`biller`) | Quando la visualizzazione del soggetto fatturante è abilitata nell'account e il payload non invia `biller`, oppure lo invia vuoto (`null`, `{}` o solo con valori vuoti). Un `biller` compilato nel payload prevale sempre |
| Altre preimpostazioni (`insurance`, `measurement`, `type`, valute, consegna residenziale, non impilabile, indirizzo del mittente) | **Non hanno effetto**: questi campi sono obbligatori nel manifesto e il valore del payload prevale sempre |

**Determinazione del responsabile dell'imposta (`tax`):**

| Configurazione dell'account | `tax` effettivamente usato |
|---|---|
| Senza preimpostazione dell'imposta | Il valore inviato nel payload |
| Predefinito `sender` o `receiver` | Il predefinito dell'account, **ignorando** il valore inviato |
| Predefinito + paese di destinazione nell'elenco delle eccezioni dell'account | L'**opposto** del predefinito (`sender` ↔ `receiver`) |
| Modalità bloccata | Sempre `sender`, indipendentemente dal payload e dalle eccezioni |

> Il paese di destinazione usato nelle eccezioni è l'`address_receiver.country_code` del payload. Se il destinatario
> è indicato solo tramite `address_code`, le eccezioni per paese non vengono valutate e vale il predefinito dell'account.
> Anche quando la preimpostazione sostituisce il valore, `tax` resta obbligatorio nel payload e deve essere `sender` o `receiver`.

### Fasi dell'elaborazione e cosa succede in caso di errore

| Fase | In caso di errore |
|---|---|
| 1. Validazione dell'input | `422`. Non viene creato nulla |
| 2. Regole dell'account e completamento (pagamento, canale, `slug`, indirizzi, scatole, articoli e kit registrati, stato) | `400`/`404`. Non viene creato nulla |
| 3. Quotazione, registrazione della spedizione e invio al corriere | Tutto viene annullato: nessuna quotazione o spedizione resta registrata nell'account |
| 4. Generazione dell'etichetta e dei documenti (dopo che il corriere ha accettato l'invio) | La spedizione **resta creata**, con numero di tracciamento, e riceve un evento di errore di elaborazione nel tracciamento. Recupera i documenti in [Documenti della Spedizione](api-documentos.md) e non ripetere il manifesto |

## Endpoint: creare un manifesto

- **Metodo e rotta:** `POST /v1/manifest`
- **Quando usarlo:** per creare, in un'unica chiamata, una spedizione già inviata al corriere e
  ricevere l'etichetta nella risposta.

### Parametri — query

| Nome | Posizione | Tipo | Obbligatorio | Predefinito | Valori | Descrizione |
|---|---|---|---|---|---|---|
| `format` | query | string | no | vuoto (tutti) | `pdf`, `zpl` | Formato dell'etichetta nella risposta. Vuoto (o valore non riconosciuto) restituisce PDF **e** ZPL |

> `format` può essere inviato anche **nel corpo** del payload. Quando viene inviato in entrambi i posti,
> **prevale il valore del corpo**. Nel corpo, i valori diversi da `pdf`/`zpl` vengono rifiutati con `422`.

### Parametri — radice del payload

| Campo | Tipo | Obbligatorio | Min. | Max. | Valori / Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `object` | string | **sì** | - | - | `doc`, `package`, `pallet`, `not_doc` | Che cosa viene spedito. `not_doc` è accettato per compatibilità e trattato come `package` |
| `type` | string | **sì** | - | - | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalità dell'invio (valore esatto, con maiuscole e spazi) |
| `tax` | string | **sì** | - | - | `sender`, `receiver` | Chi paga le imposte. Può essere sostituito dalla preimpostazione dell'account (vedi [Preimpostazioni](#preimpostazioni-dellaccount-nel-manifesto)) |
| `insurance` | boolean | **sì** | - | - | - | Stipulare un'assicurazione. Può essere attivata automaticamente se l'account ha la stipula automatica dell'assicurazione e il valore dichiarato raggiunge il minimo configurato |
| `residential_delivery` | boolean | **sì** | - | - | - | Consegna a un indirizzo residenziale |
| `non_stackable` | boolean | **sì** | - | - | - | Colli non impilabili |
| `measurement` | string | **sì** | - | - | `metric`, `imperial` | Sistema di misura di pesi e dimensioni |
| `ior` | boolean | **sì** | - | - | - | Importer of Record |
| `currency_quote` | string | **sì** | 3 | 3 | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta della quotazione (e dei valori degli articoli) |
| `currency_payment` | string | **sì** | 3 | 3 | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta del pagamento |
| `address_sender` | object | **sì** | - | - | - | Mittente (vedi tabella) |
| `address_receiver` | object | **sì** | - | - | - | Destinatario (vedi tabella) |
| `boxes` | array | **sì** | 1 elemento | - | - | Colli (vedi tabella) |
| `payment` | object | **sì** | - | - | - | Pagamento (vedi tabella) |
| `items` | array | condizionale | - | - | - | Obbligatorio quando `object` **non** è `doc` e nessuna scatola usa `kit`. **Vietato (la chiave non può esistere) quando `object` è `doc`** |
| `carrier_code` | integer | no | - | - | - | Contratto preferito (vedi [Come viene scelto il corriere](#come-viene-scelto-il-corriere)) |
| `slug` | string | no | - | 20 | - | Identificatore testuale che fissa il contratto. Deve esistere; un contratto rimosso o fuori validità restituisce `400` |
| `channel_code` | integer | no | - | - | - | Codice del canale di vendita registrato **nel tuo account** |
| `channel_identifier_number` | string | no | - | - | - | Il tuo identificatore commerciale (es.: numero dell'ordine). **Consigliato** |
| `cost_center_code` | integer | no | - | - | Preimpostazione dell'account | Centro di costo registrato per il tuo account o per account associati autorizzati. Vuoto/assente → usa la preimpostazione dell'account |
| `nfkey` | string | no | 44 | 44 | - | Chiave della NF-e (esattamente 44 caratteri) |
| `description` | string | no | 5 | 155 | - | Descrizione libera del manifesto |
| `declared_freight` | object | no | - | - | Preimpostazione dell'account | Nolo dichiarato (vedi tabella) |
| `biller` | object | no | - | - | Preimpostazione dell'account | Soggetto fatturante mostrato sull'invoice (vedi sezione) |
| `manufacturer` | object | no | - | - | - | Produttore mostrato sull'invoice (vedi sezione) |
| `extra` | object | no | - | - | - | Dati ausiliari in coppie chiave/valore (vedi tabella) |
| `format` | string | no | - | - | `pdf`, `zpl` | Alternativa alla query string (prevale su di essa) |
| `size` | string | no | - | - | `label`, `letter` | Accettato nella validazione, ma **attualmente non modifica** l'etichetta generata |

### Parametri — `address_sender` e `address_receiver`

A differenza della spedizione v1/v2, qui l'indirizzo viene indicato **completo**, perché la quotazione e l'invio
avvengono nella stessa chiamata. Ci sono due modi di indicare ogni indirizzo — **non mescolarli nello stesso oggetto**.

**Forma A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questa forma | Codice di un indirizzo registrato e visibile per il tuo account |

> Con `address_code`, **tutti** i dati dell'indirizzo provengono dall'anagrafica; qualsiasi altro campo inviato nello stesso
> oggetto viene scartato. Se invii `address_code` insieme a `country_code`, `street`, `name`, `email`,
> `phone`, `type` e `foreign`, la richiesta viene rifiutata con `422`. Mittente e destinatario non possono usare lo
> stesso `address_code`.

**Forma B — indirizzo manuale:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `country_code` | string | **sì** | 2 | 2 | - | Paese in ISO 3166-1 alpha-2 |
| `postal_code` | string | condizionale | - | 12 | - | CAP/ZIP. Obbligatorio se `state` e `state_code` non vengono indicati. In Brasile, viene usato per ricavare lo stato quando `state_code` non viene inviato |
| `state` | string | condizionale | - | 100 | - | Nome dello stato. Obbligatorio se `state_code` e `postal_code` non vengono indicati |
| `state_code` | string | condizionale | 1 | 10 | - | Sigla dello stato. Obbligatoria se `state` e `postal_code` non vengono indicati. **Consigliata sempre** (vedi le regole di business) |
| `city` | string | condizionale | 1 | 100 | - | Città. Obbligatoria se `postal_code` non viene indicato. **Consigliata sempre**: i corrieri usano la città sull'etichetta |
| `street` | string | **sì** | 5 | 70 | - | Via |
| `number` | string | no | 1 | 6 | - | Numero civico |
| `district` | string | no | 1 | 30 | - | Quartiere |
| `complement` | string | no | 1 | 20 | - | Complemento |
| `name` | string | **sì** | 1 | 70 | - | Nome del referente/ragione sociale |
| `email` | string | **sì** | - | 60 | - | E-mail valida |
| `phone` | string | no | 1 | 25 | - | Telefono. **Consigliato sempre**: i corrieri richiedono un telefono di contatto |
| `phone_extension` | string | no | - | 5 | - | Interno |
| `type` | string | **sì** | - | - | - | `pf` (persona fisica) o `pj` (persona giuridica) |
| `foreign` | boolean | **sì** | - | - | - | Indica se la persona è straniera (usato per convalidare il `federal_tax_id`) |
| `federal_tax_id` | string | condizionale | - | 20 | - | Documento federale (CPF/CNPJ o equivalente). **Obbligatorio quando `country_code` è `BR` o `AR`** |
| `state_tax_id` | string | no | - | 20 | - | Iscrizione statale |
| `description` | string | no | 1 | 155 | - | Etichetta interna dell'indirizzo |
| `company_name` | string | no | 5 | 50 | - | **Solo in `address_sender`.** Ragione sociale usata negli invii FedEx (il corriere considera solo i primi 35 caratteri) |

#### Indirizzo manuale: invia tutte le chiavi

Nel manifesto, le chiavi seguenti devono **esistere** nell'oggetto dell'indirizzo manuale, anche quando non hai
un valore per esse — in tal caso, invia `null`. L'assenza di una di esse interrompe l'elaborazione con un errore
(`500`, o `400` nella fase del corriere).

| Chiavi che devono essere presenti (il valore può essere `null`) |
|---|
| `postal_code`, `city`, `number`, `district`, `complement`, `phone`, `phone_extension`, `federal_tax_id`, `state_tax_id` |

> Non inviare nemmeno `"address_code": null` nella forma B: la chiave `address_code` deve essere omessa.

### Parametri — `boxes[]`

Ogni scatola può essere indicata in tre modi — **non mescolare le forme nella stessa scatola**:

| Forma | Campi | Comportamento |
|---|---|---|
| A — scatola registrata | `box_code` **o** `box_identifier` | Nome, dimensioni e peso provengono dall'[anagrafica delle scatole](api-caixa-v1.md) |
| B — scatola manuale | `name`, `height`, `width`, `depth` (+ `weight`/`price` in `doc`) | Dati indicati nel payload |
| C — kit | `kit` | La scatola **e** gli articoli di quella posizione provengono dal kit registrato |

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `kit` | string | no | - | 50 | SKU del kit registrato. Compila la scatola e gli articoli di quella posizione; `weight` e `price` della scatola vengono ignorati |
| `box_code` | integer | condizionale | - | - | Codice della scatola registrata. Obbligatorio quando la scatola non indica nessun altro campo |
| `box_identifier` | string | condizionale | - | 50 | Identificatore della scatola registrata. Alternativa al `box_code` |
| `name` | string | **sì** senza `kit`/`box_code`/`box_identifier` | 5 | 155 | Nome della scatola |
| `height` | number | **sì** senza `kit`/`box_code`/`box_identifier` | 1 | - | Altezza (cm in `metric`, in in `imperial`) |
| `width` | number | **sì** senza `kit`/`box_code`/`box_identifier` | 1 | - | Larghezza |
| `depth` | number | **sì** senza `kit`/`box_code`/`box_identifier` | 1 | - | Profondità |
| `weight` | number | **sì** quando `object` è `doc` (anche con scatola registrata) | 0.001 | - | Peso del collo (kg in `metric`, lb in `imperial`) |
| `price` | number | **sì** quando `object` è `doc` (anche con scatola registrata) | 0 | - | Valore dichiarato del collo. In `doc`, il valore dichiarato della spedizione è la somma dei `price` |

> Con una scatola registrata, i campi inviati nel payload completano (e sovrascrivono) quelli dell'anagrafica.
> La scatola deve appartenere al tuo account.

### Parametri — `items[]`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `box_id` | integer | **sì** | - | - | **Posizione** della scatola nell'array `boxes` (1 = prima) |
| `quantity` | integer | **sì** | 1 | 99 | Quantità dell'articolo |
| `sku` | string | condizionale | - | 50 | SKU dell'articolo registrato ([Articoli](api-itens-v1.md)). Obbligatorio quando non viene indicato nessuno tra `name`, `country_code`, `hscode`, `weight` e `unit_value` |
| `name` | string | **sì** senza `sku` | 6 | 50 | Nome dell'articolo |
| `country_code` | string | **sì** senza `sku` | 2 | 2 | Paese di origine dell'articolo (ISO 3166-1 alpha-2) |
| `hscode` | string | **sì** senza `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **sì** senza `sku` | 0.001 | 99 | Peso unitario (kg in `metric`, lb in `imperial`) |
| `unit_value` | number | **sì** senza `sku` | maggiore di 0 | - | Valore unitario, nella valuta `currency_quote`. La validazione accetta `0`, ma la quotazione rifiuta un articolo con valore zero |
| `selling_unit_value` | number | no | 0 | - | Valore di vendita unitario, usato quando `ior` è `true`. Se omesso, proviene dall'anagrafica dell'articolo (con `sku`) oppure, in sua mancanza, assume il `unit_value` |
| `description` | string | no | 5 | 450 | Descrizione dettagliata dell'articolo |
| `ean` | string | no | - | 50 | Codice EAN (codice a barre) dell'articolo. Usato come identificatore standardizzato del produttore nelle informazioni regolatorie per destinazioni nell'Unione Europea (e come identificatore del prodotto quando non c'è `sku`) |
| `nsmp` | string | no | - | 50 | *Non Standard Manufacturer Product Id*: identificatore di prodotto non standardizzato del produttore. Viene stampato sull'invoice e usato nelle informazioni regolatorie richieste da alcuni corrieri per destinazioni nell'Unione Europea |

> **Articoli con `sku`:** quando l'articolo indica uno `sku` e manca uno dei campi `name`, `country_code`, `hscode`,
> `weight`, `unit_value`, `nsmp` (o `selling_unit_value` con `ior: true`), i dati mancanti vengono completati
> a partire dall'[anagrafica dell'articolo](api-itens-v1.md) (inclusi `ean` e `nsmp`). I valori inviati nel payload
> prevalgono sempre sull'anagrafica. Se lo SKU non è registrato, l'articolo viene accettato solo se riporta
> `name`, `country_code`, `hscode` e `weight` nel payload stesso.
>
> Le informazioni regolatorie per destinazioni nell'Unione Europea vengono costruite solo quando l'articolo ha `sku` o `ean`.

> **Attenzione alle differenze rispetto alla quotazione:** qui `quantity` ha un massimo di **99** e
> `description` accetta fino a **450** caratteri.

### Parametri — `payment`

| Campo | Tipo | Obbligatorio | Valori | Descrizione |
|---|---|---|---|---|
| `payment` | object | **sì** | - | Involucro |
| `payment.type` | string | **sì** | **`billed`** | Il manifesto accetta **solo** il pagamento fatturato. Anche l'account deve avere il pagamento fatturato abilitato; in caso contrario, la risposta è `400` |

### Parametri — `declared_freight`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valore del nolo dichiarato. `0` è accettato |
| `currency` | string | **sì** quando `value` viene indicato | 3 | 3 | Valuta del nolo dichiarato |
| `declared` | boolean | no | - | - | Accettato, ma **senza effetto** nel manifesto: il nolo dichiarato viene considerato solo quando vengono indicati `value` **e** `currency` |

> Per dichiarare un nolo pari a zero, invia `"value": 0` con la `currency`. Senza `value` e `currency`, il nolo dichiarato
> non viene applicato (nemmeno quello della preimpostazione, se hai inviato la chiave `declared_freight`).

### Parametri — `extra`

Coppie chiave/valore ausiliarie, salvate nella spedizione.

| Regola | Dettaglio |
|---|---|
| Struttura | Oggetto **piatto** (un livello, senza oggetti o liste annidati) |
| Chiavi | Solo lettere minuscole, cifre e `_` |
| Valori | Testo, intero o decimale (booleani e `null` vengono rifiutati) |

| Chiave nota | Tipo | Regola | Descrizione |
|---|---|---|---|
| `federal_tax_provision` | number | - | Accantonamento per imposta federale |
| `state_tax_provision` | number | - | Accantonamento per imposta statale |
| `usd_rate_provision` | number | maggiore di 0 | Accantonamento per il cambio del dollaro. **Viene mantenuto solo** per gli account del programma Remessa Conforme con destinazione `BR`; negli altri casi viene scartato silenziosamente |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Parametri — `biller` e `manufacturer`

- `biller`: stesso formato descritto in [Spedizione (v1)](api-remessa-v1.md#parametri--biller-soggetto-fatturante)
  (`address_code` **o** campi manuali). Nel manifesto, `biller.name` ha da 5 a **355** caratteri.
  I campi non inviati assumono `type: pj` e `foreign: false`. Senza `biller` nel payload, vale la
  preimpostazione del soggetto fatturante dell'account (quando abilitata).
- `manufacturer`: stesso formato descritto in [Quotazione (v1)](api-cotacao-v1.md#parametri--manufacturer)
  (`address_code` **o** campi manuali). Con l'`address_code` di un indirizzo esistente, i dati provengono dall'anagrafica.

### Payload minimo

Manifesto di documento (`object: doc`) con indirizzi manuali — in questo caso `items` è vietato:

```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "number": null,
    "district": null,
    "complement": null,
    "name": "Negozio Esempio LTDA",
    "email": "contato@example.com",
    "phone": "11999990000",
    "phone_extension": null,
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": null,
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "number": null,
    "district": null,
    "complement": null,
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550000",
    "phone_extension": null,
    "type": "pf",
    "federal_tax_id": null,
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Busta documenti",
      "height": 2,
      "width": 23,
      "depth": 32,
      "weight": 0.3,
      "price": 50
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

**Variante minima con indirizzi registrati e kit** (`object: package`; `items` può essere omesso perché la scatola usa `kit`):

```json
{
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": { "address_code": 1234 },
  "address_receiver": { "address_code": 5678 },
  "boxes": [
    { "kit": "KIT-001" }
  ],
  "payment": { "type": "billed" }
}
```

### Payload massimo

Tutti i campi accettati. `slug` e `carrier_code` compaiono insieme solo a scopo illustrativo: in pratica usa l'uno
o l'altro (prevale lo `slug`). Le scatole mostrano la forma manuale e la forma registrata.

```json
{
  "format": "pdf",
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 12,
  "carrier_code": 45,
  "slug": "fedex-br",
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Manifesto del lotto di marzo",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Centro di distribuzione SP",
    "street": "Via Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Capannone 2",
    "company_name": "Negozio Esempio Commercio LTDA",
    "name": "Negozio Esempio LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami",
    "description": "Ufficio Miami",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550000",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Scatola master 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 1.5,
      "price": 90
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7891234567895",
      "nsmp": "MFR-TSH-000-S",
      "name": "Maglietta cotone P",
      "description": "Maglietta 100% cotone, manica corta, colore bianco, taglia P",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Scarpa sportiva",
      "description": "Scarpa da corsa uomo numero 44",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9
    }
  ],
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Sede centrale",
    "street": "Via Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Ufficio 1",
    "name": "Holding Esempio LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "phone_extension": "10",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabbrica Esempio LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "number": "200",
    "district": "Centro",
    "complement": "Blocco B",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "20",
    "type": "pj",
    "federal_tax_id": "11222333000181",
    "state_tax_id": "111222333444"
  },
  "payment": {
    "type": "billed"
  },
  "extra": {
    "federal_tax_provision": 10,
    "state_tax_provision": 7,
    "usd_rate_provision": 5.42
  }
}
```

### Esempio di request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "Commercial",
    "tax": "sender",
    "insurance": false,
    "residential_delivery": false,
    "non_stackable": false,
    "measurement": "metric",
    "ior": false,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Via Principale", "number": null, "district": null, "complement": null, "name": "Negozio Esempio LTDA", "email": "contato@example.com", "phone": "11999990000", "phone_extension": null, "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": null, "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "number": null, "district": null, "complement": null, "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550000", "phone_extension": null, "type": "pf", "federal_tax_id": null, "state_tax_id": null, "foreign": true},
    "boxes": [{"name": "Busta documenti", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Esempio di response

`200 OK` (senza `format`, la risposta contiene `AWB` e `AWB_ZPL`; con `format=pdf`, solo `AWB`; con `format=zpl`, solo `AWB_ZPL`):

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Test",
    "tracking_id": "271249639538",
    "tracking_packages_ids": [
      "271249639538"
    ],
    "docs": {
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Campi della risposta

| Campo | Tipo | Presenza | Descrizione |
|---|---|---|---|
| `data.shipping` | string (UUID) | sempre | **Identificatore della spedizione.** Usalo in tracciamento, documenti e pianificazione |
| `data.carrier` | string | sempre | Nome del contratto/corriere che ha elaborato l'invio |
| `data.tracking_id` | string | sempre | Numero di tracciamento principale (AWB) |
| `data.tracking_packages_ids` | array di string | quando il corriere lo comunica | Numero di tracciamento di ciascun collo, nell'ordine delle scatole |
| `data.docs` | object | sempre | Etichette generate, nel formato `nome → { url, base64 }` |
| `data.docs.AWB` | object | `format` vuoto o `pdf` | Etichetta in PDF |
| `data.docs.AWB_ZPL` | object | `format` vuoto o `zpl` | Etichetta in ZPL (stampanti termiche) |
| `data.docs.AWBPRC` / `data.docs.AWBPRCZPL` | object | account del programma Remessa Conforme, nelle importazioni (origine fuori dal Brasile e destinazione `BR`) | Etichetta specifica del programma, in PDF / ZPL |
| `data.docs.*.url` | string | sempre | Link del file. Se l'account protegge i link dei documenti, il link è **temporaneo** (valido per circa 5 minuti) |
| `data.docs.*.base64` | string | sempre | Contenuto del file in Base64 |
| `data.valorProvCBS`, `data.valorProvIBSEstadual`, `data.valorProvIBSMunicipal` | number | quando il calcolo delle imposte riporta questi accantonamenti | Accantonamenti di CBS e IBS (statale e municipale) |
| `data.serviceType` | string | invii FedEx | Servizio FedEx utilizzato |
| `data.operationalDetail` | object | invii FedEx | Dati operativi del corriere (`ursaPrefixCode`, `ursaSuffixCode`, `originLocationId`, `originLocationNumber`, `originServiceArea`, `destinationLocationId`, `destinationLocationNumber`, `destinationServiceArea`, `destinationLocationStateOrProvinceCode`, `deliveryDate`, `deliveryDay`, `commitDate`, `commitDay`, `ineligibleForMoneyBackGuarantee`, `astraPlannedServiceLevel`, `astraDescription`, `postalCode`, `stateOrProvinceCode`, `countryCode`, `airportId`, `serviceCode`, `packagingCode`, `publishedDeliveryTime`, `scac`). I campi senza valore arrivano `null` |

> I documenti complementari — invoice (invii che non sono `doc`), dichiarazione di contenuto (quando il
> mittente è `pf`), packing list e dichiarazione di prodotti non pericolosi — vengono generati in background.
> Consulta [Documenti della Spedizione](api-documentos.md) qualche istante dopo per ottenere l'elenco completo.

### Risposta di errore del corriere

Quando il corriere rifiuta l'invio, la risposta è `400` con `status: "error"` e riporta la sua risposta grezza:

```json
{
  "status": "error",
  "message": "[1234] Fedex - Test - Messaggio di errore del corriere",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Campo | Descrizione |
|---|---|
| `message` | `[codice del contratto] nome del corriere - messaggio` restituito dal corriere. Se il corriere risponde senza un messaggio riconoscibile, arriva `CARRIER SYNC - SHIPPING RETURN UNEXPECTED RESPONSE!` |
| `data` | Payload grezzo del corriere. **La struttura varia a seconda del corriere** e può arrivare `null` |

> Registra `message` e `data` nei tuoi log: è l'unica fonte del motivo tecnico del rifiuto. Un rifiuto del
> corriere annulla la quotazione e la spedizione; correggi il payload e invia un nuovo manifesto.

## Regole di business
- **Pagamento:** solo `billed`, e l'account deve avere il pagamento fatturato abilitato (da solo o insieme
  al pagamento immediato). Un altro valore in `payment.type` → `422`; account senza fatturazione → `400`.
- **Corrieri:** il manifesto quota solo i contratti FedEx, UPS e DHL del tuo account. La scelta segue
  [Come viene scelto il corriere](#come-viene-scelto-il-corriere).
- **Preimpostazioni:** applicate dopo la validazione, secondo
  [Preimpostazioni dell'account nel manifesto](#preimpostazioni-dellaccount-nel-manifesto). In evidenza: il `tax` della
  preimpostazione **sostituisce** quello inviato; il `cost_center_code` della preimpostazione compila il campo quando arriva vuoto.
- **`items` e `object: doc`:** con `object: doc`, la chiave `items` **non può** essere inviata (nemmeno vuota).
  Con qualsiasi altro `object`, `items` è obbligatorio — tranne quando una scatola usa `kit`.
- **Kit:** quando una scatola indica `kit`, la scatola e gli articoli di quella posizione provengono dall'anagrafica,
  e `boxes[].weight`/`boxes[].price` vengono ignorati.
- **Valore dichiarato:** in `doc`, è la somma di `boxes[].price`; negli altri casi, la somma di `unit_value × quantity`
  degli articoli. Un articolo con valore unitario vuoto o zero viene rifiutato nella quotazione (`400`).
- **`imperial`:** pesi e dimensioni indicati in libbre/pollici vengono convertiti in kg/cm prima della quotazione.
- **Stato (`state_code`):** deve essere una sigla di stato riconosciuta dalla piattaforma; per i paesi senza
  suddivisione in stati, invia lo stesso `country_code`. Senza `state_code`, la piattaforma cerca di ricavarlo
  (dal CEP in Brasile, dal nome in `state` o dal codice postale all'estero). Se non ci riesce per il
  mittente **o** per il destinatario, il manifesto viene rifiutato con `400` prima di qualsiasi registrazione.
- **`federal_tax_id`:** obbligatorio quando il paese dell'indirizzo è `BR` o `AR`.
- **`address_code`:** l'indirizzo deve essere visibile per il tuo account; con esso, gli altri campi
  dell'oggetto vengono ignorati. Mittente e destinatario non possono avere lo stesso `address_code`.
- **Indirizzo manuale:** invia tutte le chiavi elencate in
  [Indirizzo manuale: invia tutte le chiavi](#indirizzo-manuale-invia-tutte-le-chiavi) (usa `null` quando non c'è un valore).
- **`channel_code`:** deve esistere e appartenere al tuo account (anche quando proviene dalla preimpostazione).
- **`cost_center_code`:** deve esistere e appartenere al tuo account, all'account principale o ad account
  associati autorizzati.
- **`slug`:** deve esistere (`422`) e il contratto deve essere attivo ed entro il periodo di validità (`400`).
- **Assicurazione automatica:** se l'account stipula automaticamente l'assicurazione a partire da un valore minimo e il valore
  dichiarato (convertito in USD) raggiunge tale minimo, l'assicurazione viene stipulata anche con `insurance: false`.
- **`extra.usd_rate_provision`:** mantenuto solo per gli account del programma Remessa Conforme con destinazione `BR`.
- **`not_doc`** è accettato per compatibilità e trattato come `package`.

## Errori comuni

Gli errori di validazione (`422`) hanno `status: "warning"`, `message` uguale a
`Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!`
e `data` con la mappa `campo → messaggi`. Gli errori di regola di business hanno `status: "warning"` e non riportano `data`.

| Status | Significato / Messaggio | Come correggere |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificare il token inviato in `Authorization` |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contattare il supporto |
| `403` | `Has no permission!` | Richiedere al supporto il permesso di creare manifesti |
| `422` | Validazione dei campi (messaggi in `data`, es.: `O campo :attribute é obrigatório.`) | Correggere i campi elencati in `data` |
| `422` | `The :attribute field must be missing when :other is :value.` (in `items`) | Rimuovere la chiave `items` quando `object` è `doc` |
| `422` | `The :attribute field must be missing when :values are present.` (in `address_code`) | Usare solo `address_code` **o** solo i campi manuali |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Usare un `cost_center_code` del tuo account |
| `400` | `Tipo de pagamento não autorizado!` | Richiedere al supporto l'abilitazione del pagamento fatturato |
| `400` | `O canal especificado não foi encontrado!` | Usare un `channel_code` registrato nel tuo account (verifica anche la preimpostazione) |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | Usare lo `slug` di un contratto attivo |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Usare `address_code` diversi |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | Verificare l'`address_code` |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Correggere il `postal_code` o inviare `state_code` |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Correggere `state` o inviare `state_code`/`postal_code` |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | Inviare `state_code` in entrambi gli indirizzi |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | Verificare `box_code`/`box_identifier` |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | Registrare l'articolo o inviare i campi manuali |
| `400` | `O kit :sku é inválido!` / `A caixa do kit :sku é inválida!` / `Um ou mais itens do kit :sku são inválidos!` | Rivedere l'anagrafica del kit |
| `400` | Messaggio della quotazione (es.: `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. ...`, `Nenhum transportador encontrado para este usuário!`) | Nel manifesto, ogni errore della fase di quotazione torna come `400`. Leggere `message` e adeguare il payload |
| `400` | Messaggio della registrazione della spedizione | Leggere `message` e adeguare il payload |
| `404` | `A transportadora informada não está disponível para esta remessa!` | Usare il `carrier_code` di un contratto FedEx, UPS o DHL, oppure omettere il campo |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | Il contratto scelto non dispone di spedizione sincrona; contattare il supporto |
| `400` | `[código] transportador - mensagem` (`status: "error"`, `data` con la risposta del corriere) | Vedi [Risposta di errore del corriere](#risposta-di-errore-del-corriere) |
| `500` | Errore imprevisto (`status: "error"`) | **Prima di ripetere**, verifica in [Tracciamento](api-rastreio.md) se la spedizione è stata creata |

> ⚠️ **Non ripetere mai automaticamente un manifesto dopo un `500` o dopo un errore di rete/timeout.** Se l'errore
> si verifica dopo che il corriere ha accettato l'invio, la spedizione esiste davvero e una ripetizione genera un
> invio duplicato. Verifica prima tramite `channel_identifier_number` + `channel_code` in [Tracciamento](api-rastreio.md).

## Note finali
- Indica sempre `channel_identifier_number` (con `channel_code`): è la tua protezione contro i duplicati e il
  modo per individuare la spedizione se la risposta va persa.
- Il manifesto concentra molta responsabilità in una sola chiamata. Per le integrazioni che possono attendere,
  il flusso di [quotazione](api-cotacao-v1.md) + [spedizione](api-remessa-v1.md) è più sicuro.
- Per il flusso opposto (payload parziale, convalidato in seguito), vedi [Bozza](api-rascunho.md).
- Anagrafiche usate dal manifesto: [Scatola](api-caixa-v1.md) e [Articoli](api-itens-v1.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
