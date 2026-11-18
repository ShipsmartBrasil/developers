# Spedizione (v2)

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md) e [Quotazione (v2)](api-cotacao-v2.md)

## Panoramica
- Trasforma una quotazione **v2** in una spedizione reale.
- La differenza centrale rispetto alla [Spedizione v1](api-remessa-v1.md): qui indichi l'**indirizzo completo**
  (città, stato e codice postale), perché la quotazione v2 ha ricevuto solo il paese.
- La v2 espone **solo la creazione**. Status, documenti, annullamento, aggiornamento e pianificazione del ritiro
  usano gli endpoint `/v1/...`, che funzionano normalmente per le spedizioni create in v2.

> ⚠️ **La spedizione è un'operazione irreversibile nel percorso normale.** Dopo la creazione, entra in elaborazione
> presso il corriere.

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

## Flusso di utilizzo

```
1. POST /v2/quotation                 → quotation + carriers[]   (solo paese/stato)
2. POST /v2/shipping                  → spedizione ricevuta       (indirizzo completo)
3. GET  /v1/shipping/{shipping}/status→ seguire fino a "shipped"
4. GET  /v1/shipping/{shipping}/docs  → scaricare AWB, invoice ecc.
5. POST /v1/schedule                  → pianificare il ritiro (opzionale)
```

1. [Crea la quotazione v2](api-cotacao-v2.md) e verifica se è eleggibile per la spedizione.
2. Invia la spedizione con il `quotation`, il corriere scelto e l'indirizzo completo.
3. La creazione è **asincrona**: `200` significa "abbiamo ricevuto", non "abbiamo spedito".
4. Segui la spedizione con il [tracciamento](api-rastreio.md) e scarica i [documenti](api-documentos.md).

## Mappa degli endpoint per versione

| Operazione | Rotta |
|---|---|
| Creare la spedizione | `POST /v2/shipping` |
| Consultare lo status | `GET /v1/shipping/{shipping}/status` |
| Consultare lo status per canale | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` |
| Consultare i documenti | `GET /v1/shipping/{shipping}/docs` |
| Consultare i documenti per canale | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` |
| Aggiornare (NF-e, allegati, centro di costo) | `PUT /v1/shipping/{shipping}` |
| Rimuovere allegati | `DELETE /v1/shipping/{shipping}/docs` |
| Annullare | `DELETE /v1/shipping/{shipping}` |
| Annullare per canale | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` |
| Pianificare il ritiro | `POST /v1/schedule` |

> Non esiste `GET /v2/shipping/...` né `DELETE /v2/shipping/...`. Usa le rotte `/v1`.

## Endpoint: creare una spedizione

- **Metodo e rotta:** `POST /v2/shipping`
- **Quando usarlo:** dopo aver ottenuto una quotazione **v2** eleggibile, entro il periodo di validità.

### Parametri — radice del payload

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `quotation` | string | **sì** | - | - | - | UUID della quotazione. Deve essere stata creata in **v2** |
| `carrier_code` | integer | **sì** se `slug` non viene inviato | - | - | - | Codice del contratto (`carriers[].code` della quotazione) |
| `slug` | string | **sì** se `carrier_code` non viene inviato | - | 20 | - | Identificatore testuale del contratto (`carriers[].slug` della quotazione) |
| `address_sender` | object | **sì** | - | - | - | Mittente (vedi tabella sotto) |
| `address_receiver` | object | **sì** | - | - | - | Destinatario (vedi tabella sotto) |
| `payment` | object | **sì** | - | - | - | Modalità di pagamento |
| `type` | string | no | - | 30 | `Commercial` | Finalità della spedizione. Valori: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | no | 44 | 44 | `null` | Chiave della NF-e. **Esattamente 44 cifre** |
| `invoice_number` | string | no | 1 | 50 | codice interno | Numero della fattura/invoice |
| `shipment_reference` | string | no | 1 | 50 | ID interno | Riferimento della spedizione |
| `cost_center_code` | integer | no | - | - | `null` | Codice del centro di costo. Deve esistere ed essere abilitato per il tuo utente |
| `payment_notify_shipping_billing` | boolean | no | - | - | `false` | Notificare l'area di fatturazione sull'addebito |
| `attachments` | array | no | - | - | `null` | Documenti allegati alla spedizione |
| `biller` | object | no | - | - | preset dell'account | Soggetto fatturante mostrato sull'invoice al posto del mittente |

### Parametri — `address_sender` e `address_receiver`

**Questa è la differenza principale della v2:** città, stato e codice postale sono obbligatori qui.

**Modo A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questo modo | Codice dell'indirizzo registrato. Quando lo usi, non inviare gli altri campi |

**Modo B — indirizzo manuale:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `city` | string | **sì** | 3 | 100 | - | Città |
| `postal_code` | string | condizionale | - | 12 | - | Codice postale/ZIP. Obbligatorio se `state` non viene indicato. **Sempre obbligatorio per gli indirizzi in Brasile** |
| `state` | string | condizionale | - | - | - | Nome dello stato/provincia. Obbligatorio se `postal_code` non viene indicato |
| `state_code` | string | condizionale | - | - | - | Sigla dello stato/provincia. Obbligatoria se `state` e `postal_code` non vengono indicati |
| `street` | string | **sì** | - | 70 | - | Indirizzo (via) |
| `name` | string | **sì** | 5 | 35 | - | Nome del contatto/ragione sociale |
| `email` | string | **sì** | - | 60 | - | E-mail valida (riceve le notifiche del corriere) |
| `phone` | string | **sì** | - | 25 | - | Telefono |
| `type` | string | **sì** | - | - | `pj` | `pf` (persona fisica) o `pj` (persona giuridica) |
| `number` | string | no | - | 6 | `null` | Numero civico |
| `district` | string | no | - | 30 | `null` | Quartiere |
| `complement` | string | no | - | 20 | `null` | Complemento |
| `description` | string | no | - | 155 | `null` | Etichetta interna dell'indirizzo |
| `phone_extension` | string | no | - | 5 | `null` | Interno |
| `federal_tax_id` | string | condizionale | - | 20 | `null` | CPF/CNPJ o documento federale. **Obbligatorio quando il paese è `BR` o `AR`** |
| `state_tax_id` | string | condizionale | - | 20 | `null` | Partita IVA statale. **Obbligatoria quando il paese è `BR`** |
| `foreign` | boolean | condizionale | - | - | `false` | Indica se la persona è straniera. **Obbligatorio quando il paese è `BR` e non si usa `address_code`** |

> **Regola pratica:** indica `city` + (`postal_code` **o** `state_code`) + `street` + `name` + `email` +
> `phone` + `type`. Il paese arriva dalla quotazione.

### Parametri — `payment`

| Campo | Tipo | Obbligatorio | Valori | Descrizione |
|---|---|---|---|---|
| `payment.type` | string | **sì** | `billed`, `gateway`, `afterward` | Modello finanziario della spedizione |

| Valore | Significato |
|---|---|
| `billed` | Fatturato: l'importo entra nella tua fattura periodica |
| `gateway` | Pagamento immediato tramite gateway, nella valuta definita in `currency_payment` |
| `afterward` | Pagamento successivo, regolato fuori dal flusso automatico |

### Parametri — `attachments[]`

| Campo | Tipo | Obbligatorio | Max. | Descrizione |
|---|---|---|---|---|
| `description` | string | **sì** | 100 | Nome/descrizione del documento (usato in seguito per localizzare o rimuovere l'allegato) |
| `base64` | string | **sì** | - | Contenuto del file codificato in Base64 |

| Limite | Valore |
|---|---|
| Formati accettati | `PDF`, `XML`, `JPEG`, `PNG` |
| Dimensione per file | **5 MB** |

### Parametri — `biller` (soggetto fatturante)

Quando indicato, questo indirizzo compare sull'invoice al posto dell'indirizzo del mittente.
Invia `address_code` **oppure** i campi manuali. **Se non vuoi usare un soggetto fatturante, non inviare il campo.**

| Campo | Tipo | Min. | Max. | Descrizione |
|---|---|---|---|---|
| `address_code` | integer | - | - | Codice di un indirizzo registrato |
| `country_code` | string | 2 | 2 | Paese in ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | Nome dello stato |
| `state_code` | string | - | 10 | Sigla dello stato |
| `postal_code` | string | - | 12 | Codice postale/ZIP |
| `city` | string | 3 | 100 | Città |
| `street` | string | - | 70 | Indirizzo (via) |
| `number` | string | - | 6 | Numero civico |
| `district` | string | - | 30 | Quartiere |
| `complement` | string | - | 20 | Complemento |
| `description` | string | - | 155 | Etichetta interna |
| `name` | string | 5 | 35 | Nome/ragione sociale |
| `email` | string | - | 60 | E-mail |
| `phone` | string | - | 25 | Telefono |
| `phone_extension` | string | - | 5 | Interno |
| `type` | string | - | - | `pf` o `pj` |
| `federal_tax_id` | string | - | 20 | Documento federale |
| `state_tax_id` | string | - | 20 | Partita IVA statale |
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
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
    "state_code": "FL",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
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
    "phone": "11992494597",
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
    "type": "gateway"
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
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Via Principale", "name": "Negozio Esempio SRL", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "state_code": "FL", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

## Differenze rispetto alla v1

| Tema | v1 | v2 |
|---|---|---|
| Indirizzo nella spedizione | Indirizzo, nome, e‑mail, telefono, tipo (+ documenti fiscali) | Gli stessi campi **+ `city`, `state`/`state_code`, `postal_code`** |
| `city` | Non accettato (arriva dalla quotazione) | **Obbligatorio**, min. 3 e max. 100 caratteri |
| `biller` | Campi di località accettati | Campi di località **validati** (`city` min. 3) |
| Endpoint del ciclo | Creazione, status, documenti, aggiornamento, allegati, annullamento | **Solo creazione.** Il resto usa `/v1` |
| Origine della quotazione | `POST /v1/quotation` | `POST /v2/quotation` |

## Regole di business

### Eleggibilità della quotazione
| Condizione | Messaggio in caso di fallimento |
|---|---|
| La quotazione esiste e appartiene all'account | `Nenhuma cotação foi encontrada com os dados especificados!` |
| Quotazione non scaduta | `Esta cotação está expirada...` |
| Quotazione non usata prima | `Cotação já foi utilizada em outra remessa!` |
| Creata in **v2** | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| Non è `simple` di `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| Il contratto è nell'elenco della quotazione | `Contrato especificado não foi listado para esta cotação!` |

### Coerenza tra quotazione e spedizione
Il paese indicato nella spedizione deve essere lo stesso della quotazione:
- `O país :country é diferente do enviado na cotação!`
- `O Estado :state é diferente do enviado na cotação!`
- `A cidade :city é diferente do enviado na cotação!`

### Documenti fiscali obbligatori
| Paese dell'indirizzo | `federal_tax_id` | `state_tax_id` | `foreign` |
|---|---|---|---|
| `BR` | Obbligatorio (CPF o CNPJ valido) | Obbligatoria | Obbligatorio senza `address_code` |
| `AR` | Obbligatorio | - | - |
| Altri | Opzionale | Opzionale | Opzionale |

- Per `type: pf`, la partita IVA statale non può essere `ISENTO`/`ISENTA`.

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Regola di business (quotazione, indirizzo, pagamento, documento fiscale) | Leggere `message` e correggere lo scenario |
| `401` | Pagamento non autorizzato | Abilitare il tipo di pagamento sull'account |
| `403` | Permesso mancante per l'azione | Richiedere il permesso al supporto |
| `404` | `Contrato especificado não foi listado para esta cotação!` | Verificare `carrier_code`/`slug` |
| `422` | Validazione dei campi o quotazione di versione incompatibile | Correggere i campi elencati in `data` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- Usa la v2 quando l'indirizzo completo esiste solo al momento della spedizione.
- Tutto il ciclo operativo post-creazione si svolge con gli endpoint `/v1` — non distinguono la versione
  in cui la spedizione è stata creata.
- Per il dettaglio di ogni operazione post-creazione, vedi [Spedizione (v1)](api-remessa-v1.md),
  [Tracciamento](api-rastreio.md), [Documenti](api-documentos.md) e [Pianificazione](api-agendamento.md).
