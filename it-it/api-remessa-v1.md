# Spedizione (v1)

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md) e [Quotazione (v1)](api-cotacao-v1.md)

## Panoramica
- Trasforma una quotazione v1 in una spedizione reale: genera AWB, invoice, dichiarazione e gli altri documenti.
- È il modulo più completo del ciclo: creazione, consultazione dello status, consultazione dei documenti, aggiornamento,
  rimozione degli allegati e annullamento.
- Gli endpoint di status, documenti e annullamento valgono **anche** per le spedizioni create con la
  [Spedizione V2](api-remessa-v2.md).

> ⚠️ **La spedizione è un'operazione irreversibile nel percorso normale.** Dopo la creazione, entra in elaborazione
> presso il corriere. Le uniche modifiche successive ammesse sono eccezionali: collegare la NF-e quando non è
> stata indicata, allegare documenti che non sono stati inviati alla creazione e indicare il centro di costo.

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
1. POST /v1/quotation                 → quotation + carriers[]
2. POST /v1/shipping                  → spedizione ricevuta (asincrona)
3. GET  /v1/shipping/{shipping}/status→ seguire fino a "shipped"
4. GET  /v1/shipping/{shipping}/docs  → scaricare AWB, invoice ecc.
5. POST /v1/schedule                  → pianificare il ritiro (opzionale)
```

1. [Crea la quotazione](api-cotacao-v1.md) e verifica se è **eleggibile** per la spedizione.
2. Invia la spedizione con il `quotation` e il corriere scelto (`carrier_code` **o** `slug`).
3. La creazione è **asincrona**: la risposta `200` significa "abbiamo ricevuto", non "abbiamo spedito".
4. Consulta lo [status](api-rastreio.md) fino a quando compare in `shipped`.
5. Scarica i [documenti](api-documentos.md).
6. Se necessario, [pianifica il ritiro](api-agendamento.md).

## L'identificatore `{shipping}`

In tutti gli endpoint di consultazione/modifica, il parametro di path `{shipping}` accetta **tre formati**:

| Formato | Esempio | Nota |
|---|---|---|
| UUID della quotazione | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Forma consigliata |
| Numero dell'AWB | `271249639538` | Funziona per le spedizioni create negli **ultimi 90 giorni** |
| Chiave della NF-e | `12345678901234567890123456789012345678901234` | Esattamente 44 cifre numeriche |

Per consultare senza conservare nessuno di questi valori, usa gli endpoint **per canale**, basati sul
`channel_identifier_number` che hai indicato nella quotazione.

## Endpoint: creare una spedizione

- **Metodo e rotta:** `POST /v1/shipping`
- **Quando usarlo:** dopo aver ottenuto una quotazione v1 eleggibile, entro il periodo di validità.

### Parametri — radice del payload

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `quotation` | string | **sì** | - | - | - | UUID della quotazione. Deve essere stata creata in **v1** |
| `carrier_code` | integer | **sì** se `slug` non viene inviato | - | - | - | Codice del contratto (`carriers[].code` della quotazione) |
| `slug` | string | **sì** se `carrier_code` non viene inviato | - | 20 | - | Identificatore testuale del contratto (`carriers[].slug` della quotazione) |
| `address_sender` | object | **sì** | - | - | - | Mittente (vedi tabella sotto) |
| `address_receiver` | object | **sì** | - | - | - | Destinatario (vedi tabella sotto) |
| `payment` | object | **sì** | - | - | - | Modalità di pagamento (vedi tabella sotto) |
| `type` | string | no | - | 30 | `Commercial` | Finalità della spedizione. Valori: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | no | 44 | 44 | `null` | Chiave della NF-e. Quando indicata, deve avere **esattamente 44 cifre** |
| `invoice_number` | string | no | 1 | 50 | codice interno | Numero della fattura/invoice. Se omesso, viene usato il codice interno della spedizione |
| `shipment_reference` | string | no | 1 | 50 | ID interno | Riferimento della spedizione. Se omesso, viene usato l'ID interno della spedizione |
| `cost_center_code` | integer | no | - | - | `null` | Codice del centro di costo. Deve esistere ed essere abilitato per il tuo utente |
| `payment_notify_shipping_billing` | boolean | no | - | - | `false` | Notificare l'area di fatturazione sull'addebito della spedizione |
| `attachments` | array | no | - | - | `null` | Documenti allegati alla spedizione (vedi tabella sotto) |
| `biller` | object | no | - | - | preset dell'account | Soggetto fatturante mostrato sull'invoice al posto del mittente (vedi tabella sotto) |

> **`carrier_code` o `slug`:** invia **uno** dei due. Il contratto indicato deve essere nell'elenco
> `carriers[]` restituito dalla quotazione; in caso contrario l'API risponde `404` con
> `Contrato especificado não foi listado para esta cotação!`.

### Parametri — `address_sender` e `address_receiver`

Due modi, non mischiarli:

**Modo A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questo modo | Codice dell'indirizzo registrato. **Deve essere lo stesso indirizzo usato nella quotazione** |

**Modo B — indirizzo manuale:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
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
| `foreign` | boolean | condizionale | - | - | `false` | Indica se la persona è straniera. **Obbligatorio quando il paese è `BR` e non si usa `address_code`** — è ciò che permette di validare correttamente il `federal_tax_id` |

> Paese, stato, città e codice postale **non** vengono indicati qui: arrivano dalla quotazione.
> Se hai bisogno di indicarli nella spedizione, usa la [Spedizione V2](api-remessa-v2.md).

### Parametri — `payment`

| Campo | Tipo | Obbligatorio | Valori | Descrizione |
|---|---|---|---|---|
| `payment.type` | string | **sì** | `billed`, `gateway`, `afterward` | Modello finanziario della spedizione |

| Valore | Significato |
|---|---|
| `billed` | Fatturato: l'importo entra nella tua fattura periodica |
| `gateway` | Pagamento immediato tramite gateway, nella valuta definita in `currency_payment` |
| `afterward` | Pagamento successivo, regolato fuori dal flusso automatico |

> Il tipo deve essere **abilitato per il tuo account**. In caso contrario:
> `400 Tipo de pagamento inexistente!` o `401 Opção de pagamento não autorizado!`.

### Parametri — `attachments[]`

| Campo | Tipo | Obbligatorio | Max. | Descrizione |
|---|---|---|---|---|
| `description` | string | **sì** | 100 | Nome/descrizione del documento (usato in seguito per localizzare o rimuovere l'allegato) |
| `base64` | string | **sì** | - | Contenuto del file codificato in Base64 |

| Limite | Valore |
|---|---|
| Formati accettati | `PDF`, `XML`, `JPEG`, `PNG` |
| Dimensione per file | **5 MB** |
| Dimensione totale della richiesta | **90 MB** |

> File con formato non valido: `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.`
> File oltre il limite: `O arquivo enviado excede o limite máximo de 5MB.`
> XML corrotto: `O arquivo XML enviado é inválido ou está corrompido.`

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
| `city` | string | - | 100 | Città |
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

> `federal_tax_id`, `state_tax_id` e `foreign` compaiono nel payload minimo perché il mittente è brasiliano.
> Per un mittente di un altro paese (tranne `AR`), questi tre campi possono essere omessi.

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
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Via Principale", "name": "Negozio Esempio SRL", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

> La risposta **non** contiene l'AWB né i documenti: la spedizione è stata solo ricevuta. Usa il
> [tracciamento](api-rastreio.md) per seguirla e i [documenti](api-documentos.md) per scaricare i file.

### Errori specifici della creazione

| Status | Messaggio | Causa / correzione |
|---|---|---|
| `400` | `Tipo de pagamento inexistente!` | `payment.type` fuori da `billed`/`gateway`/`afterward` o non abilitato |
| `400` | `Esta cotação está expirada. Porfavor, faça outra cotação...` | `valid_until` superato — rifai la quotazione |
| `400` | `Cotação já foi utilizada em outra remessa!` | Una quotazione genera **una** spedizione |
| `400` | `Cotação Simples é inelegível para remessa!` | Usa `type: advanced` o `items` nella quotazione |
| `400` | `Chave da Nota não foi informada!!` | Il contratto richiede `nfkey` per questo scenario |
| `400` | `O endereço especificado, não foi encontrado!` | `address_code` non valido o di un altro account |
| `400` | `O país :country é diferente do enviado na cotação!` | L'indirizzo della spedizione deve coincidere con quello della quotazione |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Paese `BR`/`AR` senza `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Paese `BR` senza `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ con cifra di controllo errata |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF con cifra di controllo errata |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` con `state_tax_id` esente |
| `401` | `Opção de pagamento não autorizado!` | Abilita il tipo di pagamento sul tuo account |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` fuori dall'elenco della quotazione |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | Quotazione v2 usata in `/v1/shipping` |

## Endpoint: consultare lo status della spedizione

- **Metodo e rotta:** `GET /v1/shipping/{shipping}/status`
- **Per canale:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

| Parametro | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `shipping` | path | string | **sì** | UUID della quotazione, AWB (ultimi 90 giorni) o chiave NF-e (44 cifre) |
| `channel_identifier_number` | path | string | **sì** | Identificatore commerciale indicato nella quotazione |
| `channel_code` | path | integer | no | Codice del canale di vendita |

Dettagli, formati di risposta e codici di status in [Tracciamento della Spedizione](api-rastreio.md).

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

## Endpoint: consultare i documenti

- **Metodo e rotta:** `GET /v1/shipping/{shipping}/docs`
- **Per canale:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

| Parametro | Posizione | Tipo | Obbligatorio | Predefinito | Descrizione |
|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | UUID, AWB o chiave NF-e |
| `attachments` | query | boolean | no | `true` | Includere gli allegati inviati da te |
| `embedded` | query | boolean | no | `true` | Includere i documenti incorporati (richiede un permesso specifico) |
| `base64` | query | boolean | no | `true` | Includere il contenuto in Base64 oltre alla `url` |
| `documents` | query | string | no | tutti | Elenco separato da virgole con i tipi desiderati |

Tipi accettati in `documents`: `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

Dettagli completi in [Documenti della Spedizione](api-documentos.md).

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

## Endpoint: aggiornare spedizione

- **Metodo e rotta:** `PUT /v1/shipping/{shipping}`
- **Quando usarlo:** solo in casi eccezionali — collegare la NF-e, allegare documenti mancanti
  o indicare il centro di costo.
- **Regola:** invia `nfkey` e/o `attachments` e/o `cost_center_code`. **Almeno uno** deve essere presente.

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `nfkey` | string | condizionale | 44 | 44 | Chiave della NF-e (44 cifre numeriche) |
| `cost_center_code` | integer | condizionale | - | - | Codice del centro di costo abilitato per il tuo utente |
| `attachments` | array | condizionale | - | - | File da allegare |
| `attachments[].description` | string | **sì** se `attachments` viene inviato | - | 100 | Descrizione del documento |
| `attachments[].base64` | string | **sì** se `attachments` viene inviato | - | - | Contenuto in Base64. Formati: `PDF`, `XML`, `JPEG`, `PNG`. Max. **5 MB** per file |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Fattura", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | Nessuno dei tre campi è stato inviato |
| `404` | `Remessa não encontrada!` | `{shipping}` non valido |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | La spedizione è già avanzata troppo nel flusso |

## Endpoint: rimuovere allegati

- **Metodo e rotta:** `DELETE /v1/shipping/{shipping}/docs`
- **Quando usarlo:** per cancellare un allegato inviato per errore.

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `attachments` | array | **sì** | 1 elemento | - | Elenco degli allegati da rimuovere |
| `attachments[].id` | integer | **sì** se `name` non viene inviato | 1 | - | ID dell'allegato (restituito in `GET .../docs`) |
| `attachments[].name` | string | **sì** se `id` non viene inviato | - | 100 | Descrizione esatta dell'allegato |

> Invia **`id` OPPURE `name`** per elemento, mai entrambi.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Fattura"}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!",
  "data": {
    "deleted": [
      { "id": 123456, "name": "Fattura" }
    ],
    "failed": [
      { "id": 789, "name": null, "reason": "Nenhum documento encontrado!" }
    ]
  }
}
```

> L'operazione è **parziale**: ciò che è stato rimosso compare in `deleted` e ciò che è fallito in `failed`,
> con il motivo. Controlla sempre l'array `failed`.

## Endpoint: annullare la spedizione

- **Metodo e rotta:** `DELETE /v1/shipping/{shipping}`
- **Per canale:** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **Quando usarlo:** per richiedere l'annullamento. L'elaborazione è **asincrona**.

| Campo | Posizione | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | - | UUID, AWB o chiave NF-e |
| `channel_identifier_number` | path | string | **sì** (variante per canale) | - | - | Identificatore commerciale |
| `channel_code` | path | integer | no | - | - | Codice del canale. Solo i valori **numerici** vengono interpretati come canale |
| `reason` | body | string | no | 10 | 255 | Motivo dell'annullamento, in testo libero |

> **Attenzione all'ambiguità della rotta:** `DELETE /v1/shipping/{valore}/{numero}` viene interpretato come
> **annullamento per canale**. Per annullare tramite l'identificatore della spedizione usa sempre
> `DELETE /v1/shipping/{shipping}` (un unico segmento).

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Annullamento richiesto dal cliente"}'
```

```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Remessa especificada não foi encontrada ou não é possivel mais cancelar a mesma` | Identificatore non valido o spedizione in uno status che non permette l'annullamento |
| `422` | Validazione | `reason` con meno di 10 o più di 255 caratteri |

> Dopo aver richiesto l'annullamento, consulta lo [status](api-rastreio.md): passa per `cancelling`
> e, quando confermato dal corriere, va in `cancelled`.

## Regole di business

### Eleggibilità della quotazione
| Condizione | Messaggio in caso di fallimento |
|---|---|
| La quotazione esiste e appartiene all'account | `Nenhuma cotação foi encontrada com os dados especificados!` |
| Quotazione non scaduta | `Esta cotação está expirada...` |
| Quotazione non usata prima | `Cotação já foi utilizada em outra remessa!` |
| Stessa versione dell'API | `Não é possivel finalizar, pois seu quotation pode estar inválido...` |
| Non è `simple` di `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| Il contratto è nell'elenco della quotazione | `Contrato especificado não foi listado para esta cotação!` |

### Coerenza tra quotazione e spedizione
L'indirizzo indicato nella spedizione deve essere compatibile con quello della quotazione. Le divergenze restituiscono:
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
- Il campo `foreign` esiste affinché la validazione del `federal_tax_id` venga fatta nel formato corretto
  (documento brasiliano vs. documento straniero).

### Restrizioni del corriere
- `Cotações e envios via FedEx entre Europa e Oriente Médio estão temporariamente suspensos pela transportadora.`
- Per alcuni contratti (es.: DHL con mittente persona giuridica in Brasile), la `nfkey` è obbligatoria.
  L'assenza restituisce `Chave da Nota não foi informada!!`.

### Annullamento automatico
Le spedizioni che restano **29 giorni o più** senza eventi di tracciamento in uno status eleggibile possono essere
annullate automaticamente dalla piattaforma.

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Regola di business (quotazione, indirizzo, pagamento, documento fiscale) | Leggere `message` e correggere lo scenario |
| `401` | Pagamento non autorizzato | Abilitare il tipo di pagamento sull'account |
| `403` | Permesso mancante per l'azione | Richiedere il permesso al supporto |
| `404` | Spedizione/contratto non trovato | Verificare l'identificatore e il `carrier_code`/`slug` |
| `422` | Validazione dei campi o spedizione in uno status non modificabile | Correggere i campi elencati in `data` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- Conserva sempre il `quotation`: è l'identificatore più stabile della spedizione.
- Indicare `channel_identifier_number` nella quotazione è il modo più pratico per consultare e annullare
  usando il numero del tuo stesso ordine.
- Per indicare città, stato e codice postale **nella spedizione** (e non nella quotazione), usa la
  [Spedizione V2](api-remessa-v2.md).
- Per quotazione + spedizione + documenti in **un'unica chiamata sincrona**, vedi [Manifesto](api-manifesto.md).
