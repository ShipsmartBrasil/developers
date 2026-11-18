# Bozza

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Permette di inviare una spedizione **incompleta**: mandi ciò che hai già, e la piattaforma valida in seguito.
- La bozza viene elaborata in modo **asincrono** e, quando è completa, evolve verso quotazione + spedizione
  automaticamente.
- È il flusso indicato quando l'integrazione riceve i dati dell'ordine a tappe (es.: l'ordine arriva prima
  dell'indirizzo finale o della fattura).

> **Praticamente tutti i campi sono opzionali all'invio.** La validazione rigida avviene al momento della
> spedizione. Finché mancano dati, la bozza resta con status `incomplete` e puoi completarla.

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

## Bozza vs. altri flussi

| Tema | Quotazione + Spedizione | Manifesto | Bozza |
|---|---|---|---|
| Payload | Completo in 2 fasi | Completo in 1 chiamata | **Parziale, in quante fasi serve** |
| Validazione | Sincrona, per fase | Sincrona, totale | **Asincrona, alla spedizione** |
| Risposta | Quotazione con i prezzi | Documenti pronti | Conferma di ricezione |
| Pagamento | `billed`, `gateway`, `afterward` | `billed` | **`billed` o `afterward`** |
| Correzione successiva | Nuova quotazione | No | **Sì, tramite `PUT`** |

## Ciclo di vita della bozza

```
POST /v1/draft
     │
     ▼
  received  ──►  validating  ──►  incomplete  ──┐
                      │                          │ PUT /v1/draft/{code}
                      ▼                          │
                   complete  ◄───────────────────┘
                      │
                      ▼
                 processing  ──►  quotazione + spedizione
                      │
                      └──►  error
```

| Status | Significato | Si può modificare? |
|---|---|---|
| `received` | Bozza ricevuta, in attesa di validazione | No |
| `validating` | In validazione | No |
| `incomplete` | **Mancano dati.** Il motivo è in `status_descricao` | **Sì** |
| `edited` | Bozza aggiornata tramite `PUT`, in attesa di una nuova validazione | No |
| `complete` | Tutti i dati presenti, pronta per la spedizione | No |
| `processing` | In elaborazione: generazione di quotazione e spedizione | No |
| `error` | Errore nell'elaborazione | No |

> **Solo le bozze con status `incomplete` possono essere modificate.** Tentare di modificarle in un altro status
> restituisce `400 Apenas rascunhos com o status incomplete podem ser editados`.

## Flusso di utilizzo
1. `POST /v1/draft` con i dati che hai già.
2. Attendi l'elaborazione asincrona.
3. `GET /v1/draft` per vedere lo status e, se è `incomplete`, leggere il `status_descricao`.
4. `PUT /v1/draft/{code}` per completare ciò che mancava.
5. Quando diventa `complete`, la piattaforma genera la quotazione e la spedizione.
6. Segui la spedizione con il [tracciamento](api-rastreio.md) e scarica i [documenti](api-documentos.md).
7. `DELETE /v1/draft` per scartare le bozze che non verranno usate.

## Endpoint: creare una bozza

- **Metodo e rotta:** `POST /v1/draft`
- **Elaborazione:** asincrona.

### Parametri — radice del payload

Nessun campo è obbligatorio, ma **almeno uno** deve essere inviato.

| Campo | Tipo | Min. | Max. | Predefinito | Valori | Descrizione |
|---|---|---|---|---|---|---|
| `object` | string | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` | Cosa viene spedito |
| `type` | string | - | 30 | `Commercial` | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalità della spedizione |
| `tax` | string | - | - | preset dell'account | `sender`, `receiver` | Chi paga le imposte |
| `insurance` | boolean | - | - | preset dell'account | - | Attivare l'assicurazione |
| `measurement` | string | - | - | `metric` | `metric`, `imperial` | Sistema di misura |
| `ior` | boolean | - | - | `false` | - | Importer of Record |
| `residential_delivery` | boolean | - | - | `false` | - | Consegna residenziale |
| `non_stackable` | boolean | - | - | `false` | - | Colli non impilabili |
| `currency_quote` | string | 3 | 3 | preset dell'account | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta della quotazione |
| `currency_payment` | string | 3 | 3 | preset dell'account | vedi [valute](api-primeiros-passos.md#valute-accettate) | Valuta del pagamento |
| `nfkey` | string | 44 | 44 | - | - | Chiave della NF-e |
| `description` | string | - | 155 | - | - | Descrizione libera della bozza |
| `channel_code` | integer | - | - | preset dell'account | - | Codice del canale di vendita registrato |
| `channel_identifier_number` | string | - | - | - | - | Il tuo identificatore commerciale. **Consigliato** |
| `carrier_code` | integer | - | - | - | - | Contratto preferito, se disponibile nella quotazione |
| `address_sender` | object | - | - | preset dell'account | - | Mittente (vedi tabella) |
| `address_receiver` | object | - | - | - | - | Destinatario (vedi tabella) |
| `boxes` | array | - | - | - | - | Colli (vedi tabella) |
| `items` | array | - | - | - | - | Articoli (vedi tabella) |
| `payment` | object | - | - | - | - | Pagamento (vedi tabella) |
| `biller` | object | - | - | preset dell'account | - | Soggetto fatturante mostrato sull'invoice |
| `manufacturer` | object | - | - | - | - | Produttore mostrato sull'invoice |

### Parametri — `address_sender` e `address_receiver`

Tutti opzionali all'invio. La colonna "Obbligatorio alla spedizione" indica ciò che la piattaforma richiede perché la
bozza diventi `complete`.

| Campo | Tipo | Min. | Max. | Obbligatorio alla spedizione | Descrizione |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Codice di un indirizzo registrato (solo in `address_sender`) |
| `country_code` | string | 2 | 2 | **Sì** | Paese in ISO 3166-1 alpha-2 |
| `city` | string | - | 100 | **Sì** | Città |
| `postal_code` | string | - | 12 | Sì, se `state` non è indicato | Codice postale/ZIP |
| `state` | string | - | - | Sì, se `state_code`/`postal_code` non sono indicati | Nome dello stato |
| `state_code` | string | - | - | Sì, se `state`/`postal_code` non sono indicati | Sigla dello stato, validata rispetto al paese |
| `street` | string | - | 70 | **Sì** | Indirizzo (via) |
| `name` | string | 5 | 35 | **Sì** | Nome del contatto/ragione sociale |
| `email` | string | - | 60 | **Sì** | E-mail valida |
| `phone` | string | - | 25 | **Sì** | Telefono |
| `type` | string | - | - | **Sì** | `pf` o `pj` (predefinito `pj`) |
| `federal_tax_id` | string | - | 20 | Sì quando il paese è `BR` | CPF/CNPJ o documento federale |
| `state_tax_id` | string | - | 20 | Sì quando il paese è `BR` | Partita IVA statale |
| `number` | string | - | 100 | - | Numero civico |
| `district` | string | - | 30 | - | Quartiere |
| `complement` | string | - | 20 | - | Complemento |
| `description` | string | - | 155 | - | Etichetta interna dell'indirizzo |
| `phone_extension` | string | - | 5 | - | Interno |
| `foreign` | boolean | - | - | - | Indica se la persona è straniera |

### Parametri — `boxes[]`

| Campo | Tipo | Min. | Max. | Descrizione |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU del kit registrato. Compila la scatola **e** gli articoli di quella posizione alla spedizione |
| `box_code` | integer | - | - | Codice della scatola registrata ([Scatola](api-caixa-v1.md)) |
| `box_identifier` | string | - | - | Identificatore della scatola registrata |
| `name` | string | 4 | 155 | Nome della scatola |
| `height` | number | 1 | - | Altezza |
| `width` | number | 1 | - | Larghezza |
| `depth` | number | 1 | - | Profondità |
| `volume` | number | 0.001 | - | Volume (alternativa alle dimensioni, per `pallet`) |
| `weight` | number | 0.1 | - | Peso del collo. **Obbligatorio quando `object` è `doc`.** Fino a 3 decimali. Ignorato quando la scatola usa `kit` |
| `price` | number | 0 | - | Valore dichiarato. **Obbligatorio quando `object` è `doc`.** Ignorato quando la scatola usa `kit` |

### Parametri — `items[]`

| Campo | Tipo | Min. | Max. | Obbligatorio alla spedizione | Descrizione |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU dell'articolo registrato ([Articoli](api-itens-v1.md)) |
| `ean` | string | - | 50 | - | Codice EAN |
| `box_id` | integer | - | - | - | **Posizione** della scatola nell'array `boxes` (1 = prima) |
| `name` | string | 6 | 35 | Sì, senza `sku` | Nome dell'articolo |
| `description` | string | 4 | 155 | - | Descrizione dell'articolo |
| `country_code` | string | 2 | 2 | Sì, senza `sku` | Paese di origine dell'articolo |
| `hscode` | string | 6 | 10 | Sì, senza `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Sì, senza `sku` | Peso unitario |
| `quantity` | integer | 1 | 1000 | **Sì** | Quantità |
| `unit_value` | number | - | - | **Sì** | Valore unitario |
| `selling_unit_value` | number | - | - | Sì quando `ior` è `true` | Valore di vendita unitario |
| `is_groupable` | boolean | - | - | - | Permette di raggruppare l'articolo con altri |
| `height`, `width`, `depth` | number | 1 | - | - | Dimensioni dell'articolo |
| `is_foldable` | boolean | - | - | - | Articolo pieghevole (usato nel raggruppamento `smartbox`) |

> `items` diventa **obbligatorio alla spedizione** quando `object` è `package`, `pallet` o `not_doc`.

### Parametri — `payment`

| Campo | Tipo | Obbligatorio alla spedizione | Valori | Descrizione |
|---|---|---|---|---|
| `payment.type` | string | **Sì** | `billed`, `afterward` | La bozza **non accetta** `gateway` |

### Payload minimo

Il payload utile più piccolo — qualsiasi campo va bene, ma la bozza deve averne almeno uno:

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package"
}
```

Un esempio più realistico, con ciò che normalmente si ha già al momento dell'ordine:

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
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
    { "height": 20, "width": 20, "depth": 20 }
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

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
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
  "description": "Bozza dell'ordine del marketplace",
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
    }
  ],
  "biller": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "number": "100",
    "name": "Holding Esempio SRL",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabbrica Esempio SRL",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "type": "pj"
  },
  "payment": { "type": "billed" }
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
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "T-shirt cotone", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Necessário informar pelo menos um parâmetro para criar o rascunho` | Payload vuoto |

## Endpoint: elencare le bozze

- **Metodo e rotta:** `GET /v1/draft`
- **Elementi per pagina:** 20

| Campo | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `drafts` | body | object | no | Involucro del filtro |
| `drafts.drafts_ids` | body | array | no | Elenco dei codici delle bozze. **Vuoto o assente restituisce tutte** |
| `page` | query | integer | no | Pagina dell'elenco |

> Questo `GET` accetta un corpo JSON per il filtro. Se il tuo client HTTP non supporta un body nel `GET`,
> elenca tutte le bozze e filtra dalla tua parte.

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
```

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
          "object": "package",
          "type": "Commercial",
          "tax": "sender",
          "insurance": true,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "payment": { "type": "billed" },
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
          "items": [
            {
              "name": "T-shirt cotone",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9,
              "box_id": 1
            }
          ]
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender.country code é obrigatório. O campo address sender.street é obrigatório.",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
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
| `codigo` | integer | **Codice della bozza.** Usalo in `PUT /v1/draft/{code}` e in `drafts.drafts_ids` |
| `data` | object | Il payload della bozza, già unito alle preimpostazioni dell'account |
| `status` | string | Status attuale (vedi ciclo di vita) |
| `status_descricao` | string | **Quando è `incomplete`, elenca esattamente ciò che manca.** È il campo più importante per correggere la bozza |
| `created_at`, `updated_at` | string | Date di creazione e ultima modifica |

## Endpoint: aggiornare una bozza

- **Metodo e rotta:** `PUT /v1/draft/{code}`
- **Restrizione:** solo le bozze con status `incomplete`.
- **Struttura del payload:** identica a quella del `POST`.

| Parametro | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `code` | path | integer | **sì** | Codice della bozza (`codigo` nell'elenco) |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "address_sender": {
      "country_code": "BR",
      "postal_code": "09941070",
      "city": "Diadema",
      "street": "Via Principale",
      "name": "Negozio Esempio SRL",
      "email": "contato@example.com",
      "phone": "11989677465",
      "type": "pj",
      "federal_tax_id": "12345678000199",
      "state_tax_id": "123456789012"
    }
  }'
```

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso."
}
```

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | La bozza è in un altro status |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | `code` non valido o di un altro account |

## Endpoint: rimuovere le bozze

- **Metodo e rotta:** `DELETE /v1/draft`

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `drafts` | object | **sì** | Involucro |
| `drafts.drafts_ids` | array | **sì** | Elenco dei codici delle bozze da rimuovere |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> Vengono rimosse solo le bozze del tuo stesso account. Se nessun codice corrisponde, la risposta è
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Regole di business

### Validazione alla spedizione
Quando la bozza viene spedita, la piattaforma applica un insieme **rigido** di regole. Ciò che manca compare
in `status_descricao`. Riepilogo dei campi richiesti:

| Gruppo | Campi richiesti |
|---|---|
| Generale | `tax`, `insurance` |
| Mittente | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`state`, `state_code` o `postal_code`) |
| Mittente BR | `federal_tax_id` e `state_tax_id` |
| Destinatario | Gli stessi campi del mittente |
| Articoli (`object` ≠ `doc`) | `items` con `quantity` e `unit_value`; e `name`, `country_code`, `hscode`, `weight` quando non c'è `sku` |
| Scatole (`object` = `doc`) | `boxes[].weight` e `boxes[].price` |
| Pagamento | `payment.type` (`billed` o `afterward`) |

### Produttore
Se invii `manufacturer` **senza** `address_code`, i campi `name`, `country_code`, `state_code`,
`postal_code`, `city`, `street`, `email`, `phone` e `type` diventano obbligatori alla spedizione.

Per gli articoli con `hscode` che inizia per **`64`** (calzature), il `manufacturer.name` è obbligatorio.

### Preimpostazioni dell'account
Le preimpostazioni vengono unite alla bozza. **Il payload vince sempre sul preset**, con queste eccezioni:
- Indicare `address_sender.country_code` scarta il preset dell'indirizzo del mittente.
- Se l'account ha l'imposta bloccata (`tax locked`), il valore viene forzato a `sender`.
- Il preset del soggetto fatturante viene applicato solo quando invii `biller` vuoto.

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Bozza vuota, non trovata o non modificabile | Inviare almeno un campo, verificare il `code` e lo status |
| `403` | Permesso mancante per l'azione | Richiedere il permesso al supporto |
| `422` | Validazione dei campi | Correggere i campi elencati in `data` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- Il campo **`status_descricao` è la tua guida**: dice esattamente cosa manca perché la bozza avanzi.
- Indica sempre `channel_identifier_number`: è ciò che lega la bozza al tuo ordine e permette di
  consultare la spedizione in seguito.
- La bozza **non accetta** il pagamento tramite `gateway`. Per quello scenario usa
  [quotazione](api-cotacao-v1.md) + [spedizione](api-remessa-v1.md).
- Per il flusso opposto (tutto validato in una volta, documenti immediati), vedi [Manifesto](api-manifesto.md).
