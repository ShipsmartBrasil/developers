# Spedizione (v2)

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md) e [Quotazione (v2)](api-cotacao-v2.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Trasforma una **quotazione v2** in una spedizione reale.
- La differenza principale rispetto alla [Spedizione v1](api-remessa-v1.md): qui indichi l'**indirizzo completo**
  (città, stato e CAP), perché la quotazione v2 ha ricevuto solo il paese (e forse lo stato).
- L'indirizzo completo viene **validato nella spedizione** (CEP in Brasile, indirizzo internazionale negli altri paesi).
- La v2 espone **solo la creazione**. Status, documenti, cancellazione, aggiornamento e pianificazione del ritiro
  usano gli endpoint `/v1/...`, che funzionano normalmente per le spedizioni create nella v2.

> ⚠️ **La spedizione è un'operazione irreversibile nel percorso normale.** Una volta creata, entra in elaborazione
> presso il corriere.

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

## Flusso di utilizzo

```
1. POST /v2/quotation                  → quotation + carriers[]   (solo paese/stato)
2. POST /v2/shipping                   → spedizione ricevuta      (indirizzo completo)
3. GET  /v1/shipping/{shipping}/status → monitorare fino a "shipped"
4. GET  /v1/shipping/{shipping}/docs   → scaricare AWB, invoice ecc.
5. POST /v1/schedule                   → pianificare il ritiro (facoltativo)
```

1. [Crea la quotazione v2](api-cotacao-v2.md) e verifica che sia [idonea](#quotazioni-non-idonee-alla-spedizione).
2. Invia la spedizione con il `quotation`, il corriere scelto e l'indirizzo completo.
3. La creazione è **asincrona**: `200` significa "abbiamo ricevuto", non "abbiamo spedito".
4. Monitora tramite il [tracciamento](api-rastreio.md) e scarica i [documenti](api-documentos.md).

## Concetti essenziali

### Spedizione a partire da una quotazione v2
- Il **paese** di origine e di destinazione proviene sempre dalla quotazione (non viene indicato nella spedizione).
- Lo **stato** indicato nella quotazione (`state_code`) viene riutilizzato; se la quotazione non aveva lo stato, viene
  ricavato dal CEP/indirizzo inviato nella spedizione.
- **Città e CAP** vengono indicati qui e diventano validi per la spedizione.
- Una quotazione genera **una sola** spedizione.

### Mappa degli endpoint per versione

| Operazione | Rotta |
|---|---|
| Creare la spedizione | `POST /v2/shipping` |
| Consultare lo status | `GET /v1/shipping/{shipping}/status` |
| Consultare lo status per canale | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` |
| Consultare i documenti | `GET /v1/shipping/{shipping}/docs` |
| Consultare i documenti per canale | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` |
| Aggiornare (NF-e, allegati, centro di costo) | `PUT /v1/shipping/{shipping}` |
| Rimuovere allegati | `DELETE /v1/shipping/{shipping}/docs` |
| Cancellare | `DELETE /v1/shipping/{shipping}` |
| Cancellare per canale | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` |
| Pianificare il ritiro | `POST /v1/schedule` |

> Non esiste `GET /v2/shipping/...` né `DELETE /v2/shipping/...`. Usa le rotte `/v1`, documentate in
> [Spedizione (v1)](api-remessa-v1.md).

## Endpoint: creare la spedizione

- **Metodo e rotta:** `POST /v2/shipping`
- **Quando usarlo:** dopo aver ottenuto una quotazione **v2** idonea, entro il periodo di validità.

### Parametri — radice del payload

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `quotation` | string | **sì** | - | - | - | UUID della quotazione. Deve essere stata creata nella **v2** |
| `carrier_code` | integer | **sì** se `slug` non viene inviato | - | - | - | Codice del contratto (`carriers[].code` della quotazione) |
| `slug` | string | **sì** se `carrier_code` non viene inviato | - | 20 | - | Identificatore testuale del contratto (`carriers[].slug` della quotazione). Quando inviato, prevale su `carrier_code` |
| `address_sender` | object | **sì** | - | - | - | Mittente (vedi tabella sotto) |
| `address_receiver` | object | **sì** | - | - | - | Destinatario (vedi tabella sotto) |
| `payment` | object | **sì** | - | - | - | Modalità di pagamento |
| `type` | string | no | - | - | preimpostazione dell'account o `Commercial` | Finalità della spedizione. Valori: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | condizionale | 44 | 44 | `null` | Chiave della NF-e. **Esattamente 44 caratteri**. Obbligatoria in alcuni contratti |
| `invoice_number` | string | no | 1 | 50 | codice interno | Numero della fattura/invoice |
| `shipment_reference` | string | no | 1 | 50 | ID interno | Riferimento della spedizione |
| `cost_center_code` | integer | no | - | - | preimpostazione dell'account | Codice del centro di costo autorizzato per il tuo utente. Vuoto/nullo riceve il predefinito dell'account |
| `payment_notify_shipping_billing` | boolean | no | - | - | preimpostazione dell'account | Con `payment.type: afterward`, invia l'e-mail di addebito |
| `attachments` | array | no | - | - | `null` | Documenti allegati alla spedizione |
| `biller` | object | no | - | - | preimpostazione dell'account | Soggetto fatturante mostrato nell'invoice al posto del mittente |

### Parametri — `address_sender` e `address_receiver`

**Questa è la differenza principale della v2:** città e CAP (o stato) sono obbligatori qui.

**Forma A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questa forma | Codice dell'indirizzo registrato. Quando usato, non inviare gli altri campi |

> ⚠️ Nella v2, usa `address_code` nella spedizione **solo se lo stesso `address_code` è stato usato nella quotazione v2**.
> Una quotazione fatta solo con il paese non ha la città, e l'indirizzo registrato verrà rifiutato con
> `406 A cidade :city é diferente do enviado na cotação!`.

**Forma B — indirizzo manuale:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `city` | string | **sì** | 3 | 100 | - | Città |
| `postal_code` | string | condizionale | - | 12 | - | CEP/ZIP. Obbligatorio se `state` non viene indicato. **Sempre consigliato; in Brasile è ciò che permette di validare l'indirizzo** |
| `state` | string | condizionale | - | - | - | Nome dello stato/provincia. Obbligatorio se `postal_code` non viene indicato |
| `state_code` | string | no | - | - | stato della quotazione | Sigla dello stato/provincia. Quando la quotazione aveva `state_code`, viene usato al posto di questo |
| `street` | string | **sì** | - | 70 | - | Via |
| `name` | string | **sì** | 5 | 35 | - | Nome del contatto/ragione sociale |
| `email` | string | **sì** | - | 60 | - | E-mail valida (riceve le notifiche del corriere) |
| `phone` | string | **sì** | - | 25 | - | Telefono |
| `type` | string | **sì** | - | - | - | `pf` (persona fisica) o `pj` (persona giuridica) |
| `number` | string | no | - | 6 | `null` | Numero civico |
| `district` | string | no | - | 30 | `null` | Quartiere |
| `complement` | string | no | - | 20 | `null` | Complemento |
| `description` | string | no | - | 155 | `null` | Etichetta interna dell'indirizzo |
| `phone_extension` | string | no | - | 5 | `null` | Interno telefonico |
| `federal_tax_id` | string | condizionale | - | 20 | `null` | CPF/CNPJ o documento federale. Obbligatorio a seconda del paese (vedi [Documenti fiscali](api-remessa-v1.md#documenti-fiscali-obbligatori)) |
| `state_tax_id` | string | condizionale | - | 20 | `null` | Iscrizione statale. Obbligatoria per i brasiliani (`BR` con `foreign: false`) |
| `foreign` | boolean | no | - | - | `false` se il paese è `BR`; `true` negli altri | Indica se la persona è straniera |

> **Regola pratica:** indica `city` + `postal_code` + `street` + `name` + `email` + `phone` + `type`
> (+ documenti fiscali quando richiesti). Il paese proviene dalla quotazione.
> Senza `postal_code`, il campo `state` (nome dello stato) diventa obbligatorio — `state_code` da solo non basta.

### Parametri — `payment`

| Campo | Tipo | Obbligatorio | Valori | Descrizione |
|---|---|---|---|---|
| `payment.type` | string | **sì** | `billed`, `gateway`, `afterward` | Modello finanziario della spedizione |

| Valore | Significato | Requisito dell'account |
|---|---|---|
| `billed` | Fatturato: l'importo entra nella tua fattura periodica | Opzione di pagamento "fatturato" abilitata |
| `gateway` | Pagamento immediato tramite gateway, nella valuta definita in `currency_payment` | Opzione di pagamento "immediato" abilitata |
| `afterward` | Pagamento posticipato, regolato al di fuori del flusso automatico | - |

### Parametri — `attachments[]`

| Campo | Tipo | Obbligatorio | Max. | Descrizione |
|---|---|---|---|---|
| `description` | string | **sì** | 100 | Nome/descrizione del documento (usato in seguito per individuare o rimuovere l'allegato) |
| `base64` | string | **sì** | - | Contenuto del file codificato in Base64 |

| Limite | Valore |
|---|---|
| Formati accettati | `PDF`, `XML`, `JPEG`, `PNG` |
| Dimensione totale della richiesta | **90 MB** |

> Come nella v1, gli allegati in formato non riconosciuto vengono **scartati silenziosamente** durante la creazione. Per allegare
> con validazione, usa `PUT /v1/shipping/{shipping}` (limite di 5 MB per file).

### Parametri — `biller` (soggetto fatturante)

Quando indicato, questo indirizzo compare nell'invoice al posto dell'indirizzo del mittente.
Invia `address_code` **oppure** i campi manuali. **Se non userai un soggetto fatturante, non inviare il campo.**

| Campo | Tipo | Min. | Max. | Descrizione |
|---|---|---|---|---|
| `address_code` | integer | - | - | Codice di indirizzo registrato |
| `country_code` | string | 2 | 2 | Paese in ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | Nome dello stato |
| `state_code` | string | - | 10 | Sigla dello stato |
| `postal_code` | string | - | 12 | CEP/ZIP |
| `city` | string | 3 | 100 | Città |
| `street` | string | - | 70 | Via |
| `number` | string | - | 6 | Numero civico |
| `district` | string | - | 30 | Quartiere |
| `complement` | string | - | 20 | Complemento |
| `description` | string | - | 155 | Etichetta interna |
| `name` | string | 5 | 35 | Nome/ragione sociale |
| `email` | string | - | 60 | E-mail |
| `phone` | string | - | 25 | Telefono |
| `phone_extension` | string | - | 5 | Interno telefonico |
| `type` | string | - | - | `pf` o `pj` |
| `federal_tax_id` | string | - | 20 | Documento federale |
| `state_tax_id` | string | - | 20 | Iscrizione statale |
| `foreign` | boolean | - | - | Indica se è straniero |

### Payload minimo

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Via Principale",
    "name": "Negozio Esempio SRL",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

### Payload massimo

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "slug": "dhl-br",
  "type": "Commercial",
  "nfkey": "12345678901234567890123456789012345678901234",
  "invoice_number": "NF-000123",
  "shipment_reference": "REF-2026-0001",
  "cost_center_code": 1,
  "payment_notify_shipping_billing": true,
  "address_sender": {
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
    "phone": "13055550100",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "foreign": true
  },
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
  "payment": {
    "type": "afterward"
  },
  "attachments": [
    {
      "description": "Fattura",
      "base64": "JVBERi0xLjcKJcTl8uXrp..."
    },
    {
      "description": "XML della Fattura",
      "base64": "PD94bWwgdmVyc2lvbj0i..."
    }
  ]
}
```

### Esempio di request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Via Principale", "name": "Negozio Esempio SRL", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success` quando la spedizione è stata registrata |
| `message` | string | Conferma di ricezione. L'identificatore della spedizione è lo stesso `quotation` |

## Differenze rispetto alla v1

| Tema | v1 | v2 |
|---|---|---|
| Indirizzo nella spedizione | Via, nome, e‑mail, telefono, tipo (+ documenti fiscali) | Gli stessi campi **+ `city`, `postal_code` (o `state`)** |
| `city` | Ignorato (proviene dalla quotazione) | **Obbligatorio**, min. 3 e max. 100 caratteri |
| Validazione dell'indirizzo | Effettuata nella quotazione | **Effettuata nella spedizione** |
| `address_code` | Deve corrispondere all'indirizzo della quotazione | Funziona solo se anche la quotazione v2 ha usato `address_code` |
| `biller` | Campi di località accettati | Campi di località **validati** (`city` min. 3) |
| Endpoint del ciclo | Creazione, status, documenti, aggiornamento, allegati, cancellazione | **Solo creazione.** Il resto usa `/v1` |
| Origine della quotazione | `POST /v1/quotation` | `POST /v2/quotation` |

## Quotazioni non idonee alla spedizione

Valgono le stesse verifiche della v1, nello stesso ordine — vedi la tabella completa in
[Spedizione (v1)](api-remessa-v1.md#quotazioni-non-idonee-alla-spedizione). L'unica differenza è la versione:

| Verifica | Status | Messaggio |
|---|---|---|
| `quotation` esiste ed è stato creato nella **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| Quotazione semplice di pacco/pallet | `403` | `Cotação Simples é inelegível para remessa!` |
| Quotazione già usata | `406` | `Cotação já foi utilizada em outra remessa!` |
| Quotazione scaduta | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| Contratto fuori dall'elenco della quotazione | `404` | `Contrato especificado não foi listado para esta cotação!` |

## Regole di business

### Validazione dell'indirizzo completo
Dopo le verifiche della quotazione, l'indirizzo manuale (Forma B) viene validato:

| Caso | Status | Messaggio |
|---|---|---|
| Brasile, senza stato nella quotazione: CEP inesistente o di un altro stato | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| Fuori dal Brasile: indirizzo non individuato | `404` | `Endereço internacional enviado está errado ou não existe!` |
| Fuori dal Brasile: stato/provincia non riconosciuto | `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |

> Invia sempre `postal_code` e, quando lo conosci, `state_code` (preferibilmente già nella quotazione): è ciò che offre il maggiore
> tasso di successo nella validazione.

### Coerenza tra quotazione e spedizione
- Il paese è sempre quello della quotazione.
- Lo `state_code` della quotazione, quando esiste, prevale su quello inviato nella spedizione.
- Con `address_code`, l'indirizzo registrato deve avere lo stesso paese, stato e città registrati nella quotazione:
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Documenti fiscali, corriere e preimpostazioni
Identici alla v1:
[Documenti fiscali obbligatori](api-remessa-v1.md#documenti-fiscali-obbligatori),
[Restrizioni del corriere](api-remessa-v1.md#restrizioni-del-corriere) e
[Preimpostazioni applicate nella spedizione](api-remessa-v1.md#preimpostazioni-applicate-nella-spedizione).

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Documento fiscale, CEP brasiliano non valido, NF-e obbligatoria | Leggere `message` e correggere lo scenario |
| `401` | Pagamento non autorizzato | Abilitare il tipo di pagamento nell'account |
| `403` | Quotazione semplice/scaduta, o senza permesso | Rifare la quotazione / richiedere il permesso al supporto |
| `404` | Contratto fuori dalla quotazione, indirizzo internazionale non individuato | Verificare `carrier_code`/`slug` e l'indirizzo |
| `406` | Quotazione già usata o `address_code` incompatibile | Fare una nuova quotazione / usare un indirizzo manuale |
| `422` | Validazione dei campi, quotazione di un'altra versione, account solo DDP | Correggere i campi elencati in `data` |
| `500` | Errore imprevisto o stato/provincia non riconosciuto | Rivedere l'indirizzo; ripetere con backoff; se persiste, contattare il supporto |

## Note finali
- Usa la v2 quando l'indirizzo completo esiste solo al momento della spedizione.
- Tutto il ciclo operativo successivo alla creazione viene eseguito tramite gli endpoint `/v1` — non distinguono la versione
  in cui la spedizione è stata creata.
- Per il dettaglio di ogni operazione successiva alla creazione, vedi [Spedizione (v1)](api-remessa-v1.md),
  [Tracciamento](api-rastreio.md), [Documenti](api-documentos.md) e [Pianificazione](api-agendamento.md).
- Differenze tra le versioni in [Quotazione & Spedizione: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
