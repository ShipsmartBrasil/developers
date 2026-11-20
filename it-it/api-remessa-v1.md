# Spedizione (v1)

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md) e [Quotazione (v1)](api-cotacao-v1.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Trasforma una **quotazione v1** in una spedizione reale: genera AWB, invoice, dichiarazione e gli altri documenti.
- È il modulo più completo del ciclo: creazione, consultazione dello status, consultazione dei documenti, aggiornamento,
  rimozione degli allegati e cancellazione.
- Gli endpoint di status, documenti, aggiornamento e cancellazione servono **anche** le spedizioni create con la
  [Spedizione V2](api-remessa-v2.md).
- La creazione è **asincrona**: l'API registra la spedizione e l'invio al corriere avviene subito dopo.

> ⚠️ **La spedizione è un'operazione irreversibile nel percorso normale.** Una volta creata, entra in elaborazione
> presso il corriere. Le uniche modifiche successive ammesse sono eccezionali: collegare la NF-e quando non
> è stata indicata, allegare documenti che non sono stati inviati alla creazione e indicare/cambiare il centro di costo.

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
1. POST /v1/quotation                  → quotation + carriers[]
2. POST /v1/shipping                   → spedizione ricevuta (asincrona)
3. GET  /v1/shipping/{shipping}/status → monitorare fino a "shipped"
4. GET  /v1/shipping/{shipping}/docs   → scaricare AWB, invoice ecc.
5. POST /v1/schedule                   → pianificare il ritiro (facoltativo)
```

1. [Crea la quotazione v1](api-cotacao-v1.md) e verifica che sia [idonea](#quotazioni-non-idonee-alla-spedizione).
2. Invia la spedizione con il `quotation` e il corriere scelto (`carrier_code` **o** `slug`).
3. La creazione è **asincrona**: la risposta `200` significa "abbiamo ricevuto", non "abbiamo spedito".
4. Consulta lo [status](api-rastreio.md) finché non compare in `shipped`.
5. Scarica i [documenti](api-documentos.md).
6. Se necessario, [pianifica il ritiro](api-agendamento.md).

## Concetti essenziali

### Spedizione a partire da una quotazione v1
- La quotazione v1 ha già fornito **paese, città, stato e CAP** di origine e destinazione. Per questo, nella spedizione v1
  indichi solo **chi** spedisce e **chi** riceve (nome, contatto, via, documenti fiscali).
- I campi di località (`city`, `state`, `state_code`, `postal_code`, `country_code`) inviati in
  `address_sender`/`address_receiver` vengono **ignorati** nella v1: la località è sempre quella della quotazione.
- Una quotazione genera **una sola** spedizione.

### L'identificatore `{shipping}`

Negli endpoint di documenti, aggiornamento, rimozione degli allegati e cancellazione, il parametro di path `{shipping}`
accetta **tre formati**:

| Formato | Esempio | Osservazione |
|---|---|---|
| UUID della quotazione | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Forma consigliata. Qualsiasi valore con trattino viene trattato come UUID |
| Chiave della NF-e | `12345678901234567890123456789012345678901234` | Esattamente 44 cifre numeriche |
| Numero dell'AWB | `271249639538` | Spedizioni create negli **ultimi 90 giorni** |

> L'endpoint di **status** accetta solo UUID o AWB. Dettagli in [Tracciamento della Spedizione](api-rastreio.md).

Per consultare senza conservare nessuno di questi valori, usa gli endpoint **per canale**, basati sul
`channel_identifier_number` che hai indicato nella quotazione.

## Endpoint: creare la spedizione

- **Metodo e rotta:** `POST /v1/shipping`
- **Quando usarlo:** dopo aver ottenuto una quotazione v1 idonea, entro il periodo di validità.

### Parametri — radice del payload

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
| `quotation` | string | **sì** | - | - | - | UUID della quotazione. Deve essere stata creata nella **v1** |
| `carrier_code` | integer | **sì** se `slug` non viene inviato | - | - | - | Codice del contratto (`carriers[].code` della quotazione) |
| `slug` | string | **sì** se `carrier_code` non viene inviato | - | 20 | - | Identificatore testuale del contratto (`carriers[].slug` della quotazione). Quando inviato, prevale su `carrier_code` |
| `address_sender` | object | **sì** | - | - | - | Mittente (vedi tabella sotto) |
| `address_receiver` | object | **sì** | - | - | - | Destinatario (vedi tabella sotto) |
| `payment` | object | **sì** | - | - | - | Modalità di pagamento (vedi tabella sotto) |
| `type` | string | no | - | - | preimpostazione dell'account o `Commercial` | Finalità della spedizione. Valori: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | condizionale | 44 | 44 | `null` | Chiave della NF-e. Quando indicata, deve avere **esattamente 44 caratteri**. Obbligatoria in alcuni contratti (vedi [Regole di business](#restrizioni-del-corriere)) |
| `invoice_number` | string | no | 1 | 50 | codice interno | Numero della fattura/invoice. Se omesso, usa il codice interno della spedizione |
| `shipment_reference` | string | no | 1 | 50 | ID interno | Riferimento della spedizione. Se omesso, usa l'ID interno della spedizione |
| `cost_center_code` | integer | no | - | - | preimpostazione dell'account | Codice del centro di costo. Deve esistere ed essere autorizzato per il tuo utente. Vuoto/nullo riceve il centro di costo predefinito dell'account |
| `payment_notify_shipping_billing` | boolean | no | - | - | preimpostazione dell'account | Con `payment.type: afterward`, invia l'e-mail di addebito della spedizione |
| `attachments` | array | no | - | - | `null` | Documenti allegati alla spedizione (vedi tabella sotto) |
| `biller` | object | no | - | - | preimpostazione dell'account | Soggetto fatturante mostrato nell'invoice al posto del mittente (vedi tabella sotto) |

> **`carrier_code` o `slug`:** invia **uno** dei due. Il contratto indicato deve essere nell'elenco
> `carriers[]` restituito dalla quotazione; altrimenti l'API risponde `404` con
> `Contrato especificado não foi listado para esta cotação!`.

### Parametri — `address_sender` e `address_receiver`

Due forme, non mescolarle:

**Forma A — indirizzo già registrato:**

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `address_code` | integer | **sì** in questa forma | Codice dell'indirizzo registrato. **Deve avere lo stesso paese, stato e città della quotazione** |

**Forma B — indirizzo manuale:**

| Campo | Tipo | Obbligatorio | Min. | Max. | Predefinito | Descrizione |
|---|---|---|---|---|---|---|
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
| `federal_tax_id` | string | condizionale | - | 20 | `null` | CPF/CNPJ o documento federale. Obbligatorio a seconda del paese (vedi [Documenti fiscali](#documenti-fiscali-obbligatori)) |
| `state_tax_id` | string | condizionale | - | 20 | `null` | Iscrizione statale. Obbligatoria per i brasiliani (`BR` con `foreign: false`) |
| `foreign` | boolean | no | - | - | `false` se il paese è `BR`; `true` negli altri | Indica se la persona è straniera. Definisce come viene validato il `federal_tax_id` in Brasile |

> Il paese, lo stato, la città e il CAP **non** vengono indicati qui: provengono dalla quotazione.
> Se devi indicarli nella spedizione, usa la [Spedizione V2](api-remessa-v2.md).

### Parametri — `payment`

| Campo | Tipo | Obbligatorio | Valori | Descrizione |
|---|---|---|---|---|
| `payment.type` | string | **sì** | `billed`, `gateway`, `afterward` | Modello finanziario della spedizione |

| Valore | Significato | Requisito dell'account |
|---|---|---|
| `billed` | Fatturato: l'importo entra nella tua fattura periodica | Opzione di pagamento "fatturato" abilitata |
| `gateway` | Pagamento immediato tramite gateway, nella valuta definita in `currency_payment` | Opzione di pagamento "immediato" abilitata |
| `afterward` | Pagamento posticipato, regolato al di fuori del flusso automatico | - |

> Un valore fuori dall'elenco restituisce `422`. Un tipo non abilitato per il tuo account restituisce
> `401` con `Opção de pagamento não autorizado!`.

### Parametri — `attachments[]`

| Campo | Tipo | Obbligatorio | Max. | Descrizione |
|---|---|---|---|---|
| `description` | string | **sì** | 100 | Nome/descrizione del documento (usato in seguito per individuare o rimuovere l'allegato) |
| `base64` | string | **sì** | - | Contenuto del file codificato in Base64 |

| Limite | Valore |
|---|---|
| Formati accettati | `PDF`, `XML`, `JPEG`, `PNG` |
| Dimensione totale della richiesta | **90 MB** |

> ⚠️ **Alla creazione, gli allegati non validi vengono scartati silenziosamente:** un file in formato non riconosciuto o più grande
> del limite non genera errori — la spedizione viene creata senza di esso. Verifica gli allegati salvati in
> [Documenti](api-documentos.md). Per allegare in seguito, con validazione e messaggi di errore, usa
> [aggiornare la spedizione](#endpoint-aggiornare-la-spedizione) (limite di **5 MB** per file).

### Parametri — `biller` (soggetto fatturante)

Quando indicato, questo indirizzo compare nell'invoice al posto dell'indirizzo del mittente.
Invia `address_code` **oppure** i campi manuali. **Se non userai un soggetto fatturante, non inviare il campo** (oppure invialo vuoto:
in tal caso vale il soggetto fatturante preconfigurato nell'account, se presente).

| Campo | Tipo | Min. | Max. | Descrizione |
|---|---|---|---|---|
| `address_code` | integer | - | - | Codice di indirizzo registrato |
| `country_code` | string | 2 | 2 | Paese in ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | Nome dello stato |
| `state_code` | string | - | 10 | Sigla dello stato |
| `postal_code` | string | - | 12 | CEP/ZIP |
| `city` | string | - | 100 | Città |
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
    "street": "Via Principale",
    "name": "Negozio Esempio SRL",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  },
  "address_receiver": {
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

> `federal_tax_id` (CNPJ valido) e `state_tax_id` compaiono nel payload minimo perché il mittente è una persona giuridica
> brasiliana. Per un mittente di un altro paese (eccetto `MX` e `GR`), questi campi possono essere omessi.

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
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Via Principale", "name": "Negozio Esempio SRL", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
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
| `message` | string | Conferma di ricezione |

> La risposta **non** contiene AWB né documenti: la spedizione è stata solo ricevuta. Usa il
> [tracciamento](api-rastreio.md) per monitorarla e i [documenti](api-documentos.md) per scaricare i file.
> L'identificatore della spedizione è lo stesso `quotation`.

### Errori specifici della creazione

| Status | Messaggio | Causa / correzione |
|---|---|---|
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Documento federale assente dove è obbligatorio (vedi [Documenti fiscali](#documenti-fiscali-obbligatori)) |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brasiliano senza `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ con cifra di controllo errata |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF con cifra di controllo errata |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` in Brasile con `state_tax_id` esente |
| `400` | `Numero da nota fiscal é requerido` | Il contratto richiede `nfkey` per questo scenario |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` di un contratto rimosso/inattivo |
| `401` | `Opção de pagamento não autorizado!` | Tipo di pagamento non abilitato nel tuo account |
| `403` | `Cotação Simples é inelegível para remessa!` | Usa `type: advanced` o `items` nella quotazione |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` superato — rifai la quotazione |
| `403` | `Has no permission!` | Il tuo profilo non ha il permesso di creare spedizioni |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | `quotation` di un altro account |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` fuori dall'elenco della quotazione |
| `404` | `O endereço especificado, não foi encontrado!` | `address_code` non valido o di un altro account |
| `406` | `Cotação já foi utilizada em outra remessa!` | Una quotazione genera **una** spedizione |
| `406` | `O país :country é diferente do enviado na cotação!` | `address_code` con paese diverso da quello quotato |
| `406` | `O Estado :state é diferente do enviado na cotação!` | `address_code` con stato diverso da quello quotato |
| `406` | `A cidade :city é diferente do enviado na cotação!` | `address_code` con città diversa da quella quotata |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | `quotation` inesistente o creato nella v2 (arriva in `data.quotation`) |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente)...` | Account vincolato a DDP con quotazione DDU non pagata |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` non autorizzato per te |

> Quando c'è più di un errore di documento fiscale, i messaggi arrivano insieme in `message`, separati da ` | `.

## Endpoint: consultare lo status della spedizione

- **Metodo e rotta:** `GET /v1/shipping/{shipping}/status`
- **Per canale:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

### Parametri

| Parametro | Posizione | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|---|
| `shipping` | path | string | **sì** | UUID della quotazione o AWB |
| `channel_identifier_number` | path | string | **sì** (variante per canale) | Identificatore commerciale indicato nella quotazione |
| `channel_code` | path | integer | no | Codice del canale di vendita |

### Esempio di request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

> Formato della risposta, gruppi di status e codici in [Tracciamento della Spedizione](api-rastreio.md).

## Endpoint: consultare i documenti

- **Metodo e rotta:** `GET /v1/shipping/{shipping}/docs`
- **Per canale:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

### Parametri

| Parametro | Posizione | Tipo | Obbligatorio | Predefinito | Descrizione |
|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | UUID, chiave NF-e o AWB |
| `attachments` | query | boolean | no | `true` | Includere gli allegati inviati da te (richiede `base64=true`) |
| `embedded` | query | boolean | no | `true` | Includere i documenti incorporati (richiede un permesso specifico e `base64=true`) |
| `base64` | query | boolean | no | `true` | Includere il contenuto in Base64 oltre all'`url` |
| `documents` | query | string | no | tutti | Elenco separato da virgole con i tipi desiderati |

Tipi accettati in `documents`: `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

### Esempio di request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

> Formato della risposta e dettagli di ogni documento in [Documenti della Spedizione](api-documentos.md).

## Endpoint: aggiornare la spedizione

- **Metodo e rotta:** `PUT /v1/shipping/{shipping}`
- **Quando usarlo:** solo in casi eccezionali — collegare la NF-e, allegare documenti mancanti
  o indicare/cambiare/rimuovere il centro di costo.
- **Regola:** invia `nfkey` e/o `attachments` e/o `cost_center_code`. **Almeno uno** deve essere presente.

### Parametri

| Campo | Posizione | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | - | UUID, chiave NF-e o AWB |
| `nfkey` | body | string | condizionale | 44 | 44 | Chiave della NF-e. **Viene salvata solo se la spedizione non ha ancora una NF-e**; se ce l'ha già, viene ignorata |
| `cost_center_code` | body | integer | condizionale | - | - | Codice del centro di costo autorizzato per il tuo utente. Inviare `null` **rimuove** il centro di costo |
| `attachments` | body | array | condizionale | - | - | File da allegare |
| `attachments[].description` | body | string | **sì** se `attachments` viene inviato | - | 100 | Descrizione del documento |
| `attachments[].base64` | body | string | **sì** se `attachments` viene inviato | - | - | Contenuto in Base64. Formati: `PDF`, `XML`, `JPEG`, `PNG`. Max. **5 MB** per file |

### Payload minimo

```json
{
  "cost_center_code": 1
}
```

### Payload massimo

```json
{
  "nfkey": "12345678901234567890123456789012345678901234",
  "cost_center_code": 1,
  "attachments": [
    { "description": "Fattura", "base64": "JVBERi0xLjcK..." },
    { "description": "Foto del prodotto", "base64": "iVBORw0KGgoAAAANSUhEUg..." }
  ]
}
```

### Esempio di request

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Fattura", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success` quando l'aggiornamento è stato applicato |
| `message` | string | Conferma |

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | Nessuno dei tre campi è stato inviato |
| `400` | `Status da remessa não permite atualização na NFE.` | La spedizione ha già superato `shipped` (oppure è cancellata/in errore) |
| `400` | `Erro ao salvar arquivo!` | `base64` non valido |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Allegato superiore a 5 MB |
| `404` | `Esta remessa não foi encontrada!` | `{shipping}` non valido o di un altro account |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Spedizione cancellata o con ciclo concluso |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Formato dell'allegato non accettato |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | XML malformato |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Centro di costo non autorizzato |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Errore imprevisto |

> Le modifiche vengono applicate nell'ordine NF-e → centro di costo → allegati. Se un allegato fallisce, la NF-e e il centro
> di costo inviati nella stessa chiamata **sono già stati salvati**.

## Endpoint: rimuovere allegati

- **Metodo e rotta:** `DELETE /v1/shipping/{shipping}/docs`
- **Quando usarlo:** per eliminare un allegato inviato per errore.

### Parametri

| Campo | Posizione | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | - | UUID, chiave NF-e o AWB |
| `attachments` | body | array | **sì** | 1 elemento | - | Elenco degli allegati da rimuovere |
| `attachments[].id` | body | integer | **sì** se `name` non viene inviato | 1 | - | ID dell'allegato (restituito in `GET .../docs`) |
| `attachments[].name` | body | string | **sì** se `id` non viene inviato | - | 100 | Descrizione esatta dell'allegato |

> Invia **`id` OPPURE `name`** per elemento, mai entrambi.

### Payload minimo

```json
{
  "attachments": [
    { "id": 123456 }
  ]
}
```

### Payload massimo

```json
{
  "attachments": [
    { "id": 123456 },
    { "name": "Fattura" }
  ]
}
```

### Esempio di request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Fattura"}
    ]
  }'
```

### Esempio di response

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

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `data.deleted[]` | array | Allegati rimossi (`id`, `name`) |
| `data.failed[]` | array | Allegati non rimossi, con il motivo in `reason` |

> L'operazione è **parziale**: ciò che è stato rimosso compare in `deleted` e ciò che è fallito in `failed`,
> con il motivo. Controlla sempre l'array `failed`.

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Remessa não encontrada!` | UUID senza spedizione |
| `404` | `Esta remessa não foi encontrada!` | AWB/NF-e non individuato o di un altro account |
| `422` | Validazione | `attachments` vuoto, o elemento senza `id`/`name` |

## Endpoint: cancellare la spedizione

- **Metodo e rotta:** `DELETE /v1/shipping/{shipping}`
- **Per canale:** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **Quando usarlo:** per richiedere la cancellazione. La risposta conferma la richiesta; il completamento può essere asincrono.

### Parametri

| Campo | Posizione | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sì** | - | - | UUID, chiave NF-e, AWB **o** `channel_identifier_number` |
| `channel_identifier_number` | path | string | **sì** (variante per canale) | - | - | Identificatore commerciale |
| `channel_code` | path | integer | no | - | - | Codice del canale. Solo i valori **numerici** vengono interpretati come canale |
| `reason` | body | string | no | 10 | 255 | Motivo della cancellazione, in testo libero |

> **Come viene interpretata la rotta:**
> - `DELETE /v1/shipping/{valore}/{numero}` è sempre una **cancellazione per canale** (`channel_identifier_number` + `channel_code`).
> - `DELETE /v1/shipping/{valore}` cerca prima una spedizione il cui `channel_identifier_number` sia `{valore}`;
>   se non la trova, tratta `{valore}` come UUID, chiave NF-e o AWB.

### Payload minimo

Senza corpo:

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer IL_TUO_TOKEN'
```

### Payload massimo

```json
{
  "reason": "Il cliente ha richiesto la cancellazione del suo ordine"
}
```

### Esempio di request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/PED-2026-0001/1 \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"reason":"Il cliente ha richiesto la cancellazione del suo ordine"}'
```

### Esempio di response

```json
{
  "status": "success",
  "message": "Solicitação de cancelamento da remessa 9c06fd75-d5f4-40eb-a1a8-f12086f49944 recebida. Em breve será processada; consulte o status para acompanhar."
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | `success` quando la richiesta di cancellazione è stata accettata |
| `message` | string | Conferma, con l'UUID della spedizione |

| Status | Messaggio | Causa |
|---|---|---|
| `400` | `Esta remessa não foi encontrada!` | Identificatore non valido o spedizione di un altro account |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | Lo status attuale non consente la cancellazione (vedi tabella sotto) |
| `422` | Validazione | `reason` con meno di 10 o più di 255 caratteri; `channel_code` inesistente |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Errore imprevisto |

**Quando la cancellazione viene accettata:**

| Status attuale della spedizione | Risultato |
|---|---|
| `received`, `processing_payment`, `paid_out` | Cancellata immediatamente (`cancelled`) |
| `shipped` | Passa a `cancelling` e viene cancellata presso il corriere; alcuni contratti cancellano direttamente (`cancelled`) |
| `error` senza spedizione registrata presso il corriere | Cancellata immediatamente (`cancelled`) |
| `processing_shipment` | Accettata solo in contratti specifici; negli altri, rifiutata |
| Altri (`collecting`, `in_transit`, `delivered`, `cancelled`...) | Rifiutata con `Remessa não atende requisitos para cancelamento :id !` |

> Dopo aver richiesto la cancellazione, consulta lo [status](api-rastreio.md): passa per `cancelling`
> e, quando confermata dal corriere, diventa `cancelled`.

## Quotazioni non idonee alla spedizione

Prima di registrare la spedizione, l'API verifica la quotazione in questo ordine. La prima condizione che fallisce interrompe
la creazione:

| # | Verifica | Status | Messaggio |
|---|---|---|---|
| 1 | `quotation` esiste ed è stato creato nella **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | La quotazione appartiene al tuo account | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Account vincolato a DDP: la quotazione DDU deve essere pagata | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | Documenti fiscali di mittente e destinatario | `400` | Vedi [Documenti fiscali obbligatori](#documenti-fiscali-obbligatori) |
| 5 | Quotazione non ancora usata | `406` | `Cotação já foi utilizada em outra remessa!` |
| 6 | Non è `simple` di `package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 7 | Quotazione entro la validità | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 8 | Il contratto è nell'elenco della quotazione | `404` | `Contrato especificado não foi listado para esta cotação!` |
| 9 | Tipo di pagamento abilitato per l'account | `401` | `Opção de pagamento não autorizado!` |
| 10 | NF-e indicata quando il contratto la richiede | `400` | `Numero da nota fiscal é requerido` |

> Per sapere, **prima** di spedire, se una quotazione è già stata usata, consulta
> `GET /v1/quotation/{quotation}` e controlla `data.finished`. Dettagli degli scenari idonei in
> [Quotazione (v1)](api-cotacao-v1.md#quotazioni-non-idonee-alla-spedizione).

## Regole di business

### Coerenza tra quotazione e spedizione
- Con indirizzo manuale (Forma B), la località è sempre quella della quotazione.
- Con `address_code` (Forma A), l'indirizzo registrato deve avere **lo stesso paese, stato e città** della quotazione.
  Le divergenze restituiscono `406`:
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Documenti fiscali obbligatori
| Paese dell'indirizzo | `type` | `foreign` | `federal_tax_id` | `state_tax_id` |
|---|---|---|---|---|
| `BR` | `pf` | `false` (predefinito) | **Obbligatorio** — CPF valido | **Obbligatorio** |
| `BR` | `pj` | `false` (predefinito) | **Obbligatorio** — CNPJ valido | **Obbligatorio** |
| `BR` | qualsiasi | `true` | **Obbligatorio** (documento straniero, senza validazione di CPF/CNPJ) | Facoltativo |
| `MX`, `GR` | qualsiasi | - | **Obbligatorio** | Facoltativo |
| Altri | qualsiasi | - | Facoltativo | Facoltativo |

- Per `type: pf` in Brasile, l'iscrizione statale non può essere `ISENTO`/`ISENTA`.
- Per `type: pj` in qualsiasi paese, un `federal_tax_id` nel formato CNPJ viene validato come CNPJ.
- Con `address_code`, i documenti fiscali vengono letti dall'anagrafica dell'indirizzo e seguono le stesse regole.

### Restrizioni del corriere
- Per i contratti DHL con mittente **persona giuridica in Brasile** e oggetto diverso da documento, la `nfkey` è
  obbligatoria. La sua assenza restituisce `400` con `Numero da nota fiscal é requerido`.

### Preimpostazioni applicate nella spedizione
| Preimpostazione | Effetto |
|---|---|
| Centro di costo | Compila `cost_center_code` quando non viene inviato, o viene inviato vuoto/nullo |
| Finalità della spedizione | Compila `type` quando non viene inviato |
| Soggetto fatturante | Compila `biller` quando non viene inviato (o viene inviato vuoto) e l'account mostra il soggetto fatturante |
| Notifica di addebito | Compila `payment_notify_shipping_billing` quando non viene inviato |

### Cancellazione automatica
Le spedizioni che restano **29 giorni** senza numero di tracciamento, in uno status precedente all'invio, possono essere cancellate
automaticamente dalla piattaforma. Dettagli in [Tracciamento della Spedizione](api-rastreio.md).

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `400` | Documento fiscale, NF-e obbligatoria, contratto per `slug` non valido, cancellazione non consentita | Leggere `message` e correggere lo scenario |
| `401` | Pagamento non autorizzato | Abilitare il tipo di pagamento nell'account |
| `403` | Quotazione semplice/scaduta, o senza permesso per l'azione | Rifare la quotazione / richiedere il permesso al supporto |
| `404` | Quotazione di un altro account, spedizione/contratto/indirizzo non trovato | Verificare l'identificatore e il `carrier_code`/`slug` |
| `406` | Quotazione già usata o `address_code` incompatibile con la quotazione | Fare una nuova quotazione / usare l'indirizzo quotato |
| `422` | Validazione dei campi, quotazione di un'altra versione, account solo DDP, allegato non valido | Correggere i campi elencati in `data` |
| `500` | Errore imprevisto | Ripetere con backoff; se persiste, contattare il supporto |

## Note finali
- Conserva sempre il `quotation`: è l'identificatore più stabile della spedizione.
- Indicare `channel_identifier_number` nella quotazione è il modo più pratico per consultare e cancellare
  usando il numero del tuo stesso ordine.
- Per indicare città, stato e CAP **nella spedizione** (e non nella quotazione), usa la
  [Spedizione V2](api-remessa-v2.md). Differenze in [Quotazione & Spedizione: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Per quotazione + spedizione + documenti in **un'unica chiamata sincrona**, vedi [Manifesto](api-manifesto.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
