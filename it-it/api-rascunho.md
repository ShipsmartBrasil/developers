# Bozza

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Permette di registrare una spedizione **incompleta**: invii ciò che hai già, e la piattaforma valida in seguito.
- In ingresso, **tutti i campi sono facoltativi** (viene verificato solo il formato di ogni campo). La validazione
  rigorosa avviene dopo, nell'**elaborazione**, che viene eseguita automaticamente in cicli di pochi minuti.
- Quando la bozza supera l'elaborazione, la piattaforma genera da sola la **quotazione + spedizione** e la bozza
  **cessa di esistere** (viene rimossa). Finché mancano dati, resta `incomplete` e può essere corretta.
- È il flusso indicato quando l'integrazione riceve i dati dell'ordine in più fasi (es.: l'ordine arriva prima
  dell'indirizzo finale o della nota fiscale).
- Le **preimpostazioni dell'account** (preset) vengono applicate alla bozza, incluso il responsabile
  dell'imposta, che **sostituisce** il valore inviato quando è configurato.

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

L'header `lang` accetta `pt`, `en` e `es` (predefinito `pt`). Ogni operazione richiede un permesso specifico
nel profilo (creare, modificare, elencare e rimuovere bozze). Senza di esso, la risposta è `403 Has no permission!`.

## Flusso di utilizzo
1. `POST /v1/draft` con i dati che hai già. La risposta conferma solo la ricezione
   (**non restituisce il codice della bozza**).
2. Attendi qualche minuto: la bozza viene salvata e poi passa per l'elaborazione automatica.
3. `GET /v1/draft` per individuare la bozza (tramite il tuo `channel_identifier_number`), ottenere il `codigo`
   e vedere lo `status`. Se è `incomplete`, leggi lo `status_descricao`.
4. `PUT /v1/draft/{code}` inviando **la bozza completa e corretta** (il `PUT` sostituisce tutto il contenuto).
5. Ripeti i passi da 2 a 4 finché la bozza non scompare dall'elenco: significa che la quotazione e la spedizione sono state create.
6. Monitora la spedizione tramite il [tracciamento](api-rastreio.md) e scarica i [documenti](api-documentos.md),
   usando il `channel_identifier_number` (e il `channel_code`, se presente).
7. `DELETE /v1/draft` per scartare le bozze che non verranno usate (o che sono rimaste con status `error`).

## Concetti essenziali

### Bozza vs. altri flussi

| Tema | Quotazione + Spedizione | Manifesto | Bozza |
|---|---|---|---|
| Payload | Completo in 2 fasi | Completo in 1 chiamata | **Parziale, in tutte le fasi necessarie** |
| Validazione | Sincrona, per fase | Sincrona, totale | **Formato in ingresso; regole complete nell'elaborazione (asincrona)** |
| Risposta | Quotazione con prezzi | Documenti pronti | Conferma di ricezione (senza codice) |
| Pagamento | `billed`, `gateway`, `afterward` | `billed` | **`billed` o `afterward`** |
| Scelta del contratto | La scegli tu | La scegli tu | **Automatica** (`carrier_code` preferito o preferenza dell'account) |
| Correzione successiva | Nuova quotazione | No | **Sì, tramite `PUT` (solo in `incomplete`)** |

### Ciclo di vita della bozza

```
POST /v1/draft                      PUT /v1/draft/{code}
     │                                       │
     ▼                                       ▼
  received ──► validating ──► incomplete ──► edited ──► validating ──► ...
                    │
                    ▼
                 complete ──► processing ──► quotazione + spedizione create (bozza rimossa)
                                  │
                                  ├──► incomplete  (regola di business ha rifiutato: motivo in status_descricao)
                                  └──► error       (errore imprevisto: motivo in status_descricao)
```

| Status | Significato | Si può modificare (`PUT`)? |
|---|---|---|
| `received` | Bozza ricevuta, in attesa del prossimo ciclo di elaborazione | No |
| `edited` | Bozza aggiornata tramite `PUT`, in attesa del prossimo ciclo di elaborazione | No |
| `validating` | In validazione | No |
| `incomplete` | **Mancano dati o qualche regola ha rifiutato la spedizione.** Il motivo si trova in `status_descricao` | **Sì** |
| `complete` | Ha superato la validazione, pronta per generare quotazione e spedizione | No |
| `processing` | Generazione di quotazione e spedizione in corso | No |
| `error` | Errore imprevisto nell'elaborazione | No (rimuovila e creane una nuova) |

| Dettaglio | Comportamento |
|---|---|
| Frequenza dell'elaborazione | Le bozze `received` ed `edited` vengono validate automaticamente ogni ~3 minuti |
| Successo | La quotazione e la spedizione vengono create e **la bozza viene rimossa** (non compare più nel `GET`) |
| Bozza bloccata in `processing` | Dopo ~15 minuti senza completamento, torna a `incomplete` con `status_descricao` vuoto |
| Lunghezza di `status_descricao` | Limitata a **254 caratteri**: l'elenco delle pendenze può arrivare troncato |
| Formato di `status_descricao` | Nella validazione dei campi, riporta il primo messaggio di ogni campo, concatenati senza separatore |

> **Solo le bozze con status `incomplete` possono essere modificate.** In qualsiasi altro status la risposta è
> `400 Apenas rascunhos com o status incomplete podem ser editados`.

### Il `PUT` sostituisce l'intera bozza

Il `PUT /v1/draft/{code}` **non** si unisce al contenuto precedente: il contenuto salvato viene **sostituito** dal
payload inviato (già unito ai preset dell'account). I campi che non reinvii **vengono persi**.

- Invia sempre la bozza completa, con le correzioni.
- Attenzione quando costruisci il `PUT` a partire dal `GET`: l'elenco **non restituisce** `measurement`,
  `residential_delivery`, `non_stackable`, `cost_center_code`, `declared_freight`, `biller`,
  `manufacturer` e `address_sender.address_code`. Reinviali a partire dai tuoi dati.

### Preimpostazioni dell'account (preset) nella bozza

I preset configurati nel pannello vengono uniti al payload nel `POST` **e** nel `PUT`
(vedi [Preimpostazioni dell'account](api-primeiros-passos.md#preimpostazioni-dellaccount-presets)).

| Campo | Preset usato quando | Osservazione |
|---|---|---|
| `channel_code` | Il campo **non** viene inviato | Inviare il campo (anche `null`) impedisce il preset |
| `cost_center_code` | Il campo non viene inviato **oppure** arriva vuoto/`null` | Unico campo in cui anche un valore vuoto riceve il preset |
| `carrier_code` | Il campo non viene inviato | - |
| `tax` | **Ogni volta che esiste un preset di imposta** | Il preset **sostituisce** il valore inviato (vedi sotto) |
| `insurance` | Il campo non viene inviato | - |
| `measurement` | Il campo non viene inviato o arriva `null` | Senza preset, assume `metric` |
| `currency_quote`, `currency_payment` | Il campo non viene inviato | - |
| `residential_delivery`, `non_stackable` | Il campo non viene inviato | - |
| `type` (finalità della spedizione) | Il campo non viene inviato | - |
| `address_sender` | L'oggetto `address_sender` **non** viene inviato | Il preset entra come `address_code`. Inviare `address_sender` con qualsiasi campo scarta il preset |
| `biller` | L'oggetto non viene inviato **oppure** arriva vuoto, e l'account è configurato per mostrare il soggetto fatturante | Se invii `biller` compilato, vale sempre |
| `declared_freight` | L'oggetto non viene inviato | Nolo dichiarato trasmesso alla quotazione |

> L'unione avviene **al primo livello**: un oggetto inviato (`address_sender`, `biller`,
> `declared_freight`) sostituisce interamente l'oggetto del preset, senza combinarli campo per campo.

### Responsabile dell'imposta (`tax`)

Quando l'account ha un preset di imposta, **il valore inviato in `tax` viene ignorato** e la piattaforma determina
il responsabile in questo modo:

| Configurazione del preset di imposta | Valore salvato nella bozza |
|---|---|
| Senza preset di imposta | Il `tax` inviato. Se non lo invii, l'elaborazione segnala `tax` obbligatorio |
| Predefinito `sender` | `sender`, anche se invii `receiver` |
| Predefinito `receiver` | `receiver`, anche se invii `sender` |
| Predefinito definito e paese di destinazione nell'elenco delle eccezioni | L'**opposto** del predefinito (`sender` ↔ `receiver`) |
| Modalità vincolata | Sempre `sender` |

- Il paese di destinazione considerato è l'`address_receiver.country_code` **dello stesso payload**. Se non
  viene inviato, l'eccezione per paese non viene valutata e vale il predefinito.
- Poiché il `PUT` riapplica i preset, l'imposta viene ricalcolata a ogni aggiornamento.

## Endpoint: creare una bozza

- **Metodo e rotta:** `POST /v1/draft`
- **Quando usarlo:** per registrare una spedizione, anche con dati mancanti.
- **Elaborazione:** asincrona. La risposta `200` conferma solo la ricezione; la bozza compare nel
  `GET` qualche istante dopo.

### Parametri — radice del payload

"Ingresso" = regola verificata nella chiamata (errore `422`). "Elaborazione" = regola verificata in seguito, che lascia
la bozza `incomplete`.

| Campo | Tipo | Ingresso | Elaborazione | Min. | Max. | Predefinito | Valori / descrizione |
|---|---|---|---|---|---|---|---|
| `channel_identifier_number` | string | facoltativo | facoltativo | - | 100 | - | Il tuo identificatore dell'ordine. **Consigliato**: è tramite questo che individui la bozza e la spedizione |
| `channel_code` | integer | facoltativo | facoltativo | - | - | preimpostazione dell'account | Canale di vendita registrato. Nell'elaborazione deve appartenere al tuo account |
| `cost_center_code` | integer | facoltativo | facoltativo | - | - | preimpostazione dell'account | Centro di costo registrato del tuo account (o di account autorizzati) |
| `carrier_code` | integer | facoltativo | facoltativo | - | - | preimpostazione dell'account | Contratto preferito. Usato se è disponibile nel risultato della quotazione |
| `object` | string | facoltativo | facoltativo | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | string | facoltativo | facoltativo | - | 30 | `Commercial` (o preset) | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `tax` | string | facoltativo | **obbligatorio** | - | - | preimpostazione dell'account | `sender`, `receiver`. Vedi [Responsabile dell'imposta](#responsabile-dellimposta-tax) |
| `insurance` | boolean | facoltativo | **obbligatorio** | - | - | preimpostazione dell'account | Stipulare un'assicurazione |
| `measurement` | string | facoltativo | facoltativo | - | - | preset o `metric` | `metric`, `imperial` |
| `ior` | boolean | facoltativo | facoltativo | - | - | `false` | Importer of Record |
| `residential_delivery` | boolean | facoltativo | facoltativo | - | - | `false` (o preset) | Consegna residenziale |
| `non_stackable` | boolean | facoltativo | facoltativo | - | - | `false` (o preset) | Colli non impilabili |
| `currency_quote` | string | facoltativo | **necessario** (vedi nota) | 3 | 3 | preimpostazione dell'account | Valuta della quotazione. Vedi [valute](api-primeiros-passos.md#valute-accettate) |
| `currency_payment` | string | facoltativo | **necessario** (vedi nota) | 3 | 3 | preimpostazione dell'account | Valuta del pagamento. Vedi [valute](api-primeiros-passos.md#valute-accettate) |
| `nfkey` | string | facoltativo | facoltativo | 44 | 44 | - | Chiave della NF-e |
| `description` | string | facoltativo | facoltativo | - | 155 | - | Descrizione libera |
| `address_sender` | object | facoltativo | **obbligatorio** | - | - | preimpostazione dell'account | Mittente (vedi tabella) |
| `address_receiver` | object | facoltativo | **obbligatorio** | - | - | - | Destinatario (vedi tabella) |
| `boxes` | array | facoltativo | vedi regole | - | - | - | Colli (vedi tabella) |
| `items` | array | facoltativo | vedi regole | - | - | - | Articoli (vedi tabella) |
| `payment` | object | facoltativo | **obbligatorio** | - | - | - | Pagamento (vedi tabella) |
| `declared_freight` | object | facoltativo | facoltativo | - | - | preimpostazione dell'account | Nolo dichiarato: `value` (number) e `currency` (3 lettere). Non viene verificato in ingresso; viene trasmesso alla quotazione |
| `biller` | object | facoltativo | facoltativo | - | - | preimpostazione dell'account | Soggetto fatturante mostrato nell'invoice (stessa struttura del mittente) |
| `manufacturer` | object | facoltativo | facoltativo | - | - | - | Produttore mostrato nell'invoice (vedi tabella) |

> **Nota sulle valute:** la validazione dell'elaborazione non segnala l'assenza di `currency_quote` e
> `currency_payment`, ma senza di esse (nel payload o nel preset) la generazione della quotazione fallisce e la bozza
> passa a `error`, che **non può essere modificata**. Indica sempre entrambe le valute o configura il preset.

### Parametri — `address_sender` e `address_receiver`

Due modi per indicare il **mittente** (non mescolarli):
- **Forma A — indirizzo registrato:** invia solo `address_sender.address_code`. Nell'elaborazione, l'indirizzo
  registrato viene copiato nella bozza e validato con le stesse regole indicate sotto.
- **Forma B — campi manuali:** invia i campi sotto e **non** inviare `address_code`.

Il destinatario accetta **solo la Forma B** (`address_receiver.address_code` non viene usato).

| Campo | Tipo | Min. | Max. (ingresso) | Max. (elaborazione) | Obbligatorio nell'elaborazione | Descrizione |
|---|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | - | Codice di indirizzo registrato (solo `address_sender`) |
| `country_code` | string | 2 | 2 | 2 | **Sì** | Paese in ISO 3166-1 alpha-2 |
| `postal_code` | string | - | 12 | 12 | Sì, se `state` non viene indicato | CEP/ZIP |
| `state` | string | - | - | - | Sì, se `postal_code` e `state_code` non vengono indicati | Nome dello stato |
| `state_code` | string | - | - | - | Vedi nota | Sigla dello stato, verificata rispetto alla base degli stati. Per un paese senza stati, usa lo stesso codice del paese |
| `city` | string | - | 100 | 100 | **Sì** | Città |
| `street` | string | - | 70 | 70 | **Sì** | Via |
| `number` | string | - | 100 | **6** | - | Numero civico |
| `district` | string | - | 30 | 30 | - | Quartiere |
| `complement` | string | - | 20 | 20 | - | Complemento |
| `description` | string | - | 155 | 155 | - | Etichetta interna dell'indirizzo |
| `name` | string | 5 | 35 | 35 | **Sì** | Nome del contatto/ragione sociale |
| `email` | string | - | 60 | 60 | **Sì** | E-mail valida |
| `phone` | string | - | 25 | 25 | **Sì** | Telefono |
| `phone_extension` | string | - | 5 | 5 | - | Interno telefonico |
| `type` | string | - | - | - | **Sì** (senza predefinito) | `pf` o `pj` |
| `federal_tax_id` | string | - | 20 | 20 | Sì quando il paese è `BR` | Documento federale. Per `BR` senza `foreign: true`, deve essere un CPF o CNPJ valido |
| `state_tax_id` | string | - | 20 | 20 | Sì quando il paese è `BR` | Iscrizione statale |
| `foreign` | boolean | - | - | - | - | Indica una persona straniera (esonera dal controllo di CPF/CNPJ) |

> **Localizzazione dell'indirizzo:** indica `postal_code` **oppure** `state`. `state_code` da solo non basta,
> perché, senza `state`, il `postal_code` diventa obbligatorio.
>
> **Attenzione a `number`:** l'ingresso accetta fino a 100 caratteri, ma l'elaborazione ne accetta solo fino a **6**.
> Valori più lunghi lasciano la bozza `incomplete`.

### Parametri — `boxes[]`

Quando **invii `boxes`**, ogni scatola deve essere indicata in **uno** dei modi sotto (non mescolarli
nella stessa scatola):

| Forma | Campi | Cosa succede |
|---|---|---|
| A — kit | `kit` | La scatola e gli articoli del kit vengono compilati a partire dal kit registrato |
| B — scatola registrata per codice | `box_code` | Usa la [scatola registrata](api-caixa-v1.md) |
| C — scatola registrata per identificatore | `box_identifier` | Usa la scatola registrata |
| D — scatola manuale | `name`, `height`, `width`, `depth` | **I quattro campi sono obbligatori nell'elaborazione** |

| Campo | Tipo | Min. | Max. | Descrizione |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU del kit registrato |
| `box_code` | integer | - | - | Codice della scatola registrata |
| `box_identifier` | string | - | - | Identificatore della scatola registrata |
| `name` | string | 4 | 155 | Nome della scatola |
| `height` | number | 1 | - | Altezza |
| `width` | number | 1 | - | Larghezza |
| `depth` | number | 1 | - | Profondità |
| `volume` | number | 0.001 | - | Volume. Accettato, ma **non esonera** dalle dimensioni nell'elaborazione |
| `weight` | number | 0.1 | - | Peso del collo, fino a 3 cifre decimali. **Obbligatorio (già in ingresso) quando `object` è `doc`**. Ignorato in una scatola con `kit` |
| `price` | number | 0 | - | Valore dichiarato. **Obbligatorio (già in ingresso) quando `object` è `doc`**. Ignorato in una scatola con `kit` |

> Con `boxes` e `object` diverso da `doc`, **ogni scatola che non sia un kit deve avere almeno un articolo**
> che punta a essa in `items[].box_id`. Con `object: doc`, gli `items` vengono scartati.

### Parametri — `items[]`

Due modi per indicare ogni articolo:
- **Forma A — articolo registrato:** `sku` (+ `quantity`, `unit_value` e, con `boxes`, `box_id`). I dati
  mancanti provengono dall'[anagrafica degli articoli](api-itens-v1.md). Lo SKU deve esistere.
- **Forma B — articolo manuale:** tutti i campi indicati come obbligatori senza `sku`.

| Campo | Tipo | Min. | Max. | Obbligatorio nell'elaborazione | Descrizione |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU dell'articolo registrato |
| `ean` | string | - | 50 | - | Codice EAN |
| `nsmp` | string | - | 50 | - | *Non Standard Manufacturer Product Id*: identificatore di prodotto non standardizzato del produttore. Stampato nell'invoice e usato nelle informazioni normative richieste da alcuni corrieri per destinazioni nell'Unione Europea. Con `sku`, se non viene inviato, proviene dall'anagrafica dell'articolo |
| `box_id` | integer | - | - | Sì, quando ci sono `boxes` | **Posizione** della scatola nell'array `boxes` (1 = prima). Senza `boxes`, viene ignorato |
| `name` | string | 6 | 35 | Sì, senza `sku` | Nome dell'articolo |
| `description` | string | 4 | 155 | - | Descrizione dell'articolo (il minimo di 4 viene verificato nell'elaborazione) |
| `country_code` | string | 2 | 2 | Sì, senza `sku` | Paese di origine dell'articolo |
| `hscode` | string | 6 | 10 | Sì, senza `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Sì, senza `sku` | Peso unitario (il massimo di 99 viene verificato nell'elaborazione) |
| `quantity` | integer | 1 | 1000 | **Sì** | Quantità |
| `unit_value` | number | - | - | **Sì** | Valore unitario |
| `selling_unit_value` | number | - | - | Richiesto dalla quotazione quando `ior` è `true` | Valore di vendita unitario |
| `height`, `width`, `depth` | number | 1 | - | Sì, quando **non** ci sono `boxes` (salvo SKU registrato con dimensioni) | Dimensioni dell'articolo, usate per calcolare le scatole |
| `is_groupable` | boolean | - | - | - | Predefinito `true`. Senza `boxes`: gli articoli raggruppabili vanno insieme in una scatola calcolata; `false` genera una scatola per unità |
| `is_foldable` | boolean | - | - | - | Accettato, ma senza effetto nella bozza |

> **Quando `items` è obbligatorio:** senza `boxes`, `items` è sempre richiesto (anche per `doc`). Con
> `boxes`, è richiesto per `package`, `pallet` e `not_doc`, eccetto quando le scatole usano `kit`.
>
> **Senza `boxes`:** la piattaforma costruisce le scatole a partire dalle dimensioni degli articoli. Per `doc`, invia
> sempre `boxes` con `weight` e `price`.

### Parametri — `payment`

| Campo | Tipo | Obbligatorio nell'elaborazione | Valori | Descrizione |
|---|---|---|---|---|
| `payment.type` | string | **Sì** | `billed`, `afterward` | La bozza **non accetta** `gateway` (errore `422` in ingresso) |

### Parametri — `biller`

Stessa struttura e limiti di `address_sender`, incluso `address_code` (Forma A) o campi manuali
(Forma B). Nell'elaborazione, tutti i campi del `biller` sono facoltativi; il `number` accetta fino a 6 caratteri.
Se non invii `biller` (o lo invii vuoto), può essere applicato il soggetto fatturante del preset dell'account.

### Parametri — `manufacturer`

Invia `address_code` **oppure** i campi manuali. Il produttore **non viene verificato in ingresso**: i limiti
sotto vengono applicati nell'elaborazione.

| Campo | Tipo | Min. | Max. | Obbligatorio nell'elaborazione | Descrizione |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Codice di indirizzo registrato. Quando usato, gli altri campi sono facoltativi |
| `name` | string | 5 | 30 | Sì, senza `address_code` | Ragione sociale/nome del produttore |
| `country_code` | string | 2 | 2 | Sì, senza `address_code` | Paese |
| `state_code` | string | 1 | 10 | Sì, senza `address_code` | Sigla dello stato |
| `state` | string | - | 155 | - | Nome dello stato |
| `postal_code` | string | - | 12 | Sì, senza `address_code` | CEP/ZIP |
| `city` | string | - | 100 | Sì, senza `address_code` | Città |
| `street` | string | - | 70 | Sì, senza `address_code` | Via |
| `number` | string | - | 6 | - | Numero civico |
| `district` | string | - | 30 | - | Quartiere |
| `complement` | string | - | 20 | - | Complemento |
| `email` | string | - | 60 | Sì, senza `address_code` | E-mail |
| `phone` | string | - | 25 | Sì, senza `address_code` | Telefono |
| `phone_extension` | string | - | 6 | - | Interno telefonico |
| `type` | string | - | - | Sì, senza `address_code` | `pf` o `pj` |
| `federal_tax_id` | string | - | 20 | - | Documento federale |
| `state_tax_id` | string | - | 20 | - | Iscrizione statale |

### Payload minimo

Il payload più piccolo accettato è un oggetto vuoto: crea una bozza che resterà `incomplete` finché non verrà completata.

```json
{}
```

In pratica, invia almeno il tuo identificatore, per poter individuare la bozza in seguito:

```json
{
  "channel_identifier_number": "PED-2026-0001"
}
```

Payload minimo affinché la bozza **superi l'elaborazione** senza dipendere dai preset (mittente e destinatario
fuori dal Brasile, scatola manuale):

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "US",
    "postal_code": "10001",
    "city": "New York",
    "street": "Fifth Avenue",
    "name": "Negozio Esempio",
    "email": "contato@example.com",
    "phone": "12125550100",
    "type": "pj"
  },
  "address_receiver": {
    "country_code": "CA",
    "postal_code": "M5V2T6",
    "city": "Toronto",
    "street": "King Street",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "14165550100",
    "type": "pf"
  },
  "boxes": [
    { "name": "Scatola P", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "T-shirt cotone",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9
    }
  ],
  "payment": { "type": "billed" }
}
```

### Payload massimo

Tutti i campi accettati. I colli mostrano le forme manuale (scatola 1), scatola registrata (scatola 2)
e kit (scatola 3); in un'integrazione reale usa la forma adeguata a ciascun collo. Il mittente usa la Forma B
(campi manuali); per la Forma A, invia solo `"address_sender": {"address_code": 1}`.

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 1,
  "carrier_code": 1,
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
  "description": "Bozza ordine del marketplace",
  "declared_freight": {
    "value": 50,
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
    "name": "Negozio Esempio SRL",
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
    "phone": "11992494597",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": "987654321",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Scatola master",
      "height": 20,
      "width": 20,
      "depth": 20,
      "volume": 0.008,
      "weight": 2.5,
      "price": 150
    },
    { "box_identifier": "CX0236" },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "NSMP-TSH-000-S",
      "name": "T-shirt cotone S",
      "description": "T-shirt 100% cotone manica corta bianca",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true
    },
    {
      "box_id": 2,
      "name": "Bermuda in twill M",
      "description": "Bermuda in twill beige",
      "country_code": "BR",
      "hscode": "620342",
      "weight": 0.4,
      "quantity": 1,
      "unit_value": 59.9,
      "selling_unit_value": 89.9
    }
  ],
  "payment": { "type": "billed" },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Sede",
    "street": "Via Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Ufficio 1",
    "name": "Holding Esempio SRL",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "phone_extension": "10",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabbrica Esempio SRL",
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
    "federal_tax_id": "11222333000144",
    "state_tax_id": "111222333444"
  }
}
```

### Esempio di request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"name": "Scatola P", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "T-shirt cotone", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Esempio di response

`200 OK`

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

`422 Unprocessable Entity` (es.: `payment.type` con `gateway`)

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "payment.type": ["O campo payment.type não contém um valor válido."]
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success`, `warning` o `error` |
| `message` | string | Messaggio di conferma o di errore |
| `data` | object | Solo in caso di errore `422`: mappa campo → elenco di messaggi |

> La risposta di successo **non contiene il codice della bozza**. Usa `GET /v1/draft` per ottenerlo.

## Endpoint: elencare le bozze

- **Metodo e rotta:** `GET /v1/draft`
- **Quando usarlo:** per scoprire il `codigo` di una bozza e monitorare lo `status`/`status_descricao`.
- **Ambito:** solo le bozze del tuo account. **Elementi per pagina:** 20.

### Parametri

| Campo | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `page` | query | integer | no | Pagina dell'elenco (predefinito 1) |
| `drafts` | body | object | no | Envelope del filtro |
| `drafts.drafts_ids` | body | array | no | Codici delle bozze (interi o stringhe). **Vuoto o assente restituisce tutte** |

> Questo `GET` accetta un corpo JSON per il filtro. Se il tuo client HTTP non supporta un body nel `GET`,
> elencale tutte (paginando) e filtra dalla tua parte.

### Payload minimo

Senza corpo e senza query: elenca la prima pagina di tutte le bozze.

```
GET /v1/draft
```

### Payload massimo

```
GET /v1/draft?page=1
```

```json
{
  "drafts": {
    "drafts_ids": [30, 31]
  }
}
```

### Esempio di request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "codigo": 30,
        "codigo_usuario": 1,
        "data": {
          "channel_identifier_number": "PED-2026-0001",
          "carrier_code": "",
          "nfkey": "",
          "object": "package",
          "type": "Commercial",
          "channel_code": "",
          "channel": "",
          "tax": "sender",
          "insurance": true,
          "ior": false,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "description": "",
          "address_sender": {
            "country_code": "",
            "state": "",
            "state_code": "",
            "postal_code": "",
            "city": "",
            "description": "",
            "street": "",
            "number": "",
            "district": "",
            "complement": "",
            "name": "",
            "email": "",
            "phone": "",
            "type": "",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "address_receiver": {
            "country_code": "US",
            "state": "",
            "state_code": "",
            "postal_code": "33101",
            "city": "Miami",
            "description": "",
            "street": "Main Avenue",
            "number": "",
            "district": "",
            "complement": "",
            "name": "John Doe",
            "email": "johndoe@example.com",
            "phone": "11992494597",
            "type": "pf",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "items": [
            {
              "box_id": 1,
              "name": "T-shirt cotone",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9
            }
          ],
          "boxes": [
            { "name": "Scatola P", "height": 20, "width": 20, "depth": 20 }
          ],
          "payment": { "type": "billed" }
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "links": [
      { "url": null, "label": "« Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/draft?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo »", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/draft",
    "per_page": 20,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.data[].codigo` | integer | **Codice della bozza.** Usalo in `PUT /v1/draft/{code}` e in `drafts.drafts_ids` |
| `data.data[].codigo_usuario` | integer | Codice dell'account proprietario della bozza |
| `data.data[].data` | object | Contenuto della bozza (già unito ai preset), in formato fisso: i campi assenti arrivano come `""` |
| `data.data[].data.channel` | string | Nome del canale di vendita (quando c'è `channel_code`) |
| `data.data[].status` | string | Status attuale (vedi [ciclo di vita](#ciclo-di-vita-della-bozza)) |
| `data.data[].status_descricao` | string/null | **Quando `incomplete` o `error`, indica cosa ha impedito l'avanzamento** (fino a 254 caratteri) |
| `data.data[].created_at`, `updated_at` | string | Date nel formato `AAAA-MM-GG HH:MM:SS` |
| `data.current_page`, `last_page`, `per_page`, `total`, `from`, `to` | integer | Dati della paginazione |
| `data.first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url`, `path`, `links` | string/array | Link di navigazione tra le pagine |

> L'oggetto `data` dell'elenco **non include** `measurement`, `residential_delivery`, `non_stackable`,
> `cost_center_code`, `declared_freight`, `biller`, `manufacturer` né `address_sender.address_code`.
> Quando il mittente è stato indicato tramite `address_code`, dopo la prima elaborazione compare espanso
> con i dati dell'indirizzo registrato.
>
> Se non esiste alcuna bozza per i filtri (o la pagina richiesta è oltre l'ultima), la risposta è
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`, e non un elenco vuoto.

## Endpoint: aggiornare una bozza

- **Metodo e rotta:** `PUT /v1/draft/{code}`
- **Quando usarlo:** per correggere una bozza `incomplete`.
- **Restrizione:** solo bozze del tuo account con status `incomplete`.
- **Corpo:** stessi campi e regole del `POST`. **Il contenuto precedente viene completamente sostituito**
  (vedi [Il PUT sostituisce l'intera bozza](#il-put-sostituisce-lintera-bozza)).
- **Effetto:** lo status passa a `edited` e la bozza torna al prossimo ciclo di elaborazione.
  Lo `status_descricao` precedente rimane fino alla nuova validazione.

### Parametri

| Parametro | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `code` | path | integer | **sì** | Codice della bozza (`codigo` nell'elenco) |
| corpo | body | object | no | Stessa struttura del [`POST`](#endpoint-creare-una-bozza) |

### Payload minimo

`PUT /v1/draft/30` con corpo `{}`. È accettato, ma **cancella** il contenuto della bozza (restano solo i preset).
Usalo solo se vuoi ricominciare la bozza da zero.

```json
{}
```

### Payload massimo

Lo stesso [payload massimo del `POST`](#payload-massimo). In pratica, reinvia la bozza completa con le correzioni.

### Esempio di request

Bozza che era `incomplete` per mancanza del mittente: il corpo contiene il contenuto originale **più** il mittente.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Via Principale", "number": "100", "name": "Negozio Esempio SRL", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"name": "Scatola P", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "T-shirt cotone", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso.",
  "data": {
    "codigo": 30,
    "codigo_usuario": 1,
    "data": {
      "channel_identifier_number": "PED-2026-0001",
      "object": "package",
      "type": "Commercial",
      "tax": "sender",
      "insurance": true,
      "measurement": "metric",
      "currency_quote": "USD",
      "currency_payment": "BRL",
      "address_sender": {
        "country_code": "BR",
        "postal_code": "09941070",
        "city": "Diadema",
        "street": "Via Principale",
        "number": "100",
        "name": "Negozio Esempio SRL",
        "email": "contato@example.com",
        "phone": "11989677465",
        "type": "pj",
        "federal_tax_id": "12345678000199",
        "state_tax_id": "123456789012"
      },
      "address_receiver": {
        "country_code": "US",
        "postal_code": "33101",
        "city": "Miami",
        "street": "Main Avenue",
        "name": "John Doe",
        "email": "johndoe@example.com",
        "phone": "11992494597",
        "type": "pf"
      },
      "boxes": [
        { "name": "Scatola P", "height": 20, "width": 20, "depth": 20 }
      ],
      "items": [
        {
          "box_id": 1,
          "name": "T-shirt cotone",
          "country_code": "BR",
          "hscode": "610910",
          "weight": 0.25,
          "quantity": 3,
          "unit_value": 29.9
        }
      ],
      "payment": { "type": "billed" }
    },
    "status": "edited",
    "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
    "created_at": "2026-02-20 19:04:00",
    "updated_at": "2026-02-20 19:20:31"
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.codigo` | integer | Codice della bozza |
| `data.codigo_usuario` | integer | Codice dell'account proprietario della bozza |
| `data.data` | object | Contenuto salvato: il payload inviato già unito ai preset. Può contenere campi aggiuntivi provenienti dalle preimpostazioni dell'account; ignora quelli che non conosci |
| `data.status` | string | Sempre `edited` dopo l'aggiornamento |
| `data.status_descricao` | string/null | Motivo della validazione precedente (verrà aggiornato alla prossima elaborazione) |
| `data.created_at`, `data.updated_at` | string | Date nel formato `AAAA-MM-GG HH:MM:SS` |

## Endpoint: rimuovere bozze

- **Metodo e rotta:** `DELETE /v1/draft`
- **Quando usarlo:** per scartare le bozze che non verranno usate, incluse quelle rimaste in `error`.
- **Ambito:** vengono rimossi solo i codici che appartengono al tuo account; gli altri vengono ignorati.
  La rimozione è definitiva.

### Parametri

| Campo | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `drafts` | body | object | **sì** | Envelope |
| `drafts.drafts_ids` | body | array | **sì** (non vuoto) | Codici delle bozze da rimuovere (interi o stringhe) |

### Payload minimo

```json
{ "drafts": { "drafts_ids": [30] } }
```

### Payload massimo

```json
{ "drafts": { "drafts_ids": [30, 31, 32] } }
```

### Esempio di request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success` quando almeno una bozza dell'elenco è stata rimossa |
| `message` | string | Messaggio di conferma |

> Basta che **un** codice dell'elenco appartenga al tuo account perché la risposta sia `success`; gli altri vengono
> ignorati senza avviso. Se nessun codice corrisponde, la risposta è
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Regole di business

### Validazione nell'elaborazione
Nell'elaborazione, la piattaforma applica un insieme **rigoroso** di regole. Ciò che manca compare in
`status_descricao` e la bozza resta `incomplete`.

| Gruppo | Requisiti |
|---|---|
| Generale | `tax` e `insurance` presenti (nel payload o tramite preset); `channel_identifier_number` testo fino a 100 caratteri |
| Mittente e destinatario | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`postal_code` o `state`); `number` fino a 6 caratteri |
| Indirizzi in Brasile (`BR`) | `federal_tax_id` e `state_tax_id`; `federal_tax_id` deve essere un CPF o CNPJ valido, salvo `foreign: true` |
| Senza `boxes` | `items` obbligatorio (qualsiasi `object`), con dimensioni per calcolare le scatole |
| Con `boxes` | Ogni scatola: `kit`, `box_code`, `box_identifier` o (`name` + `height` + `width` + `depth`). Per `object` ≠ `doc`, ogni scatola (eccetto kit) ha bisogno di almeno un articolo con il suo `box_id` |
| `object` = `doc` | `boxes[].weight` e `boxes[].price` (verificati già in ingresso) |
| Articoli | `quantity` (da 1 a 1000) e `unit_value`; senza `sku`: `name`, `country_code`, `hscode`, `weight` (fino a 99). Con `sku`, lo SKU deve essere registrato |
| Pagamento | `payment.type` (`billed` o `afterward`) |
| Produttore | Con `manufacturer` senza `address_code`: `name`, `country_code`, `state_code`, `postal_code`, `city`, `street`, `email`, `phone` e `type` |
| Canale | `channel_code`, quando indicato (o proveniente dal preset), deve appartenere al tuo account |

### Generazione della quotazione e della spedizione
Una volta validata, la bozza diventa automaticamente una quotazione e una spedizione:

| Fase | Comportamento |
|---|---|
| Kit | Ogni `boxes[].kit` viene sostituito dalla scatola e dagli articoli del kit registrato |
| Scatole | Con `boxes`, usa le scatole inviate. Senza `boxes`, calcola le scatole in base alle dimensioni degli articoli (raggruppabili insieme; non raggruppabili, una scatola per unità) |
| Quotazione | Tipo semplice per `doc` e avanzato per gli altri. Usa `declared_freight` del payload o del preset |
| Scelta del contratto | Se `carrier_code` è tra i risultati, viene usato. Altrimenti, la piattaforma sceglie in base alla preferenza dell'account: il **più economico** o il **più veloce** |
| Spedizione | `type` vuoto diventa `Commercial`; `nfkey`, `cost_center_code` e `biller` vengono trasmessi quando indicati |
| Rifiuto per regola di business | La bozza torna a `incomplete`, con il messaggio della quotazione/spedizione in `status_descricao` |
| Errore imprevisto | La bozza passa a `error`. Rimuovila e creane una nuova |

### Messaggi possibili in `status_descricao`
Oltre ai messaggi di validazione dei campi (es.: `O campo address sender.country code é obrigatório.`),
possono comparire:

| Messaggio | Causa | Come correggere |
|---|---|---|
| `Nenhum item foi enviado para este rascunho` | Senza `items` (e senza kit) | Inviare `items` |
| `O item com o sku :sku não foi encontrado!` | SKU non registrato e articolo senza dati completi | Registrare l'articolo o inviare i campi manuali |
| `Nenhuma caixa foi enviado para este rascunho` | `boxes` vuoto, o `box_code`/`box_identifier` non trovato | Verificare le scatole |
| `Parâmetros da caixa não foram prenchidos corretamente por favor certifique-se dos seguintes campos (name,height,width,depth)` | Scatola manuale senza `name`, `height`, `width` o `depth` | Completare i quattro campi |
| `Nenhum item foi associado para a caixa :name !` | Scatola senza articoli con il suo `box_id` | Correggere `items[].box_id` |
| `Não há itens válidos, verifique os dados e/ou o codigo SKU` | Senza `boxes`, nessun articolo utilizzabile | Verificare gli articoli e gli SKU |
| `O canal especificado não foi encontrado!` | `channel_code` non appartiene al tuo account | Correggere il `channel_code` |
| `O kit :sku é inválido!` | Kit inesistente o inattivo | Verificare lo SKU del kit |
| `A caixa do kit :sku é inválida!` | La scatola del kit non è stata trovata | Rivedere l'anagrafica del kit |
| `Um ou mais itens do kit :sku são inválidos!` | Articolo del kit non trovato | Rivedere l'anagrafica del kit |
| Messaggi della quotazione o della spedizione | Es.: nessun contratto disponibile, restrizioni di carico | Vedi [Quotazione](api-cotacao-v1.md) e [Spedizione](api-remessa-v1.md) |

## Errori comuni

| Status | Significato / Messaggio | Come correggere |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | Modifica solo bozze `incomplete`; consulta lo status nel `GET` |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Verificare il `code`/`drafts_ids` (può essere di un altro account o essere già stato convertito in spedizione e rimosso) |
| `403` | `Has no permission!` | Richiedere al supporto il permesso per l'azione (creare, modificare, elencare o rimuovere bozze) |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + `data` con i campi | Correggere i campi elencati in `data` |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` (in `data.cost_center_code`) | Usare un centro di costo del tuo account o di account autorizzati |
| `422` | `O campo drafts é obrigatório.` / `O campo drafts.drafts ids é obrigatório.` (nel `DELETE`) | Inviare `drafts.drafts_ids` con almeno un codice |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Ripetere con backoff; se persiste, contattare il supporto |

Gli errori di autenticazione (token non valido, account bloccato) si trovano in
[Primi Passi](api-primeiros-passos.md#errori-di-autenticazione).

## Note finali
- Il campo **`status_descricao` è la tua guida**: indica cosa ha impedito alla bozza di avanzare.
- Indica sempre `channel_identifier_number`: è ciò che collega la bozza al tuo ordine, permette di trovare il
  `codigo` nell'elenco e di consultare la spedizione dopo che la bozza è stata rimossa.
- Nel `PUT`, **reinvia la bozza completa**: il contenuto precedente viene sostituito.
- Se l'account ha un preset di imposta, il `tax` inviato viene ignorato; verifica il valore salvato nel `GET`.
- La bozza **non accetta** pagamenti tramite `gateway`. Per questo scenario usa
  [quotazione](api-cotacao-v1.md) + [spedizione](api-remessa-v1.md).
- Per il flusso opposto (tutto validato in una volta, documenti immediati), vedi [Manifesto](api-manifesto.md).
- Anagrafiche usate dalla bozza: [Scatola](api-caixa-v1.md), [Articoli](api-itens-v1.md), [Contratti](api-contrato-v1.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
