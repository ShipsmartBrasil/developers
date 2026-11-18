# Quotazione & Spedizione: v1 vs v2

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
Esistono due versioni pubblicate per la coppia **quotazione + spedizione**. Usano lo **stesso motore di calcolo**
e gli **stessi corrieri**. La differenza sta in **quando** indichi l'indirizzo completo.

| | v1 | v2 |
|---|---|---|
| Indirizzo completo | Indicato **nella quotazione** | Indicato **nella spedizione** |
| Payload della quotazione | Paese + città + codice postale/stato | Solo il paese (+ stato opzionale) |
| Payload della spedizione | Indirizzo, nome, contatto | Indirizzo, nome, contatto **+ città, stato, codice postale** |
| Ciclo operativo completo | Sì, in `/v1` | Creazione in `/v2`, il resto in `/v1` |

## In una frase

> **Usa la v1** se, al momento di quotare, sai già verso quale città/codice postale invierai.
> **Usa la v2** se, al momento di quotare, conosci solo il paese di destinazione.

## Regola d'oro: non mischiare le versioni

Una quotazione creata in v1 genera **solo** una spedizione in `POST /v1/shipping`.
Una quotazione creata in v2 genera **solo** una spedizione in `POST /v2/shipping`.

Mischiarle restituisce `422`:
```
Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.
```

## Confronto della quotazione

| Tema | v1 (`POST /v1/quotation`) | v2 (`POST /v2/quotation`) |
|---|---|---|
| `address_*.country_code` | Obbligatorio | Obbligatorio |
| `address_*.city` | **Obbligatorio** | Non accettato |
| `address_*.postal_code` | Obbligatorio (o `state`) | Non accettato |
| `address_*.state` | Obbligatorio (o `state_code`/`postal_code`) | Non accettato |
| `address_*.state_code` | Obbligatorio (o `state`/`postal_code`) | **Opzionale** (migliora la precisione) |
| `object` | `doc`, `package`, `pallet`, `not_doc` | Identico |
| `type` | `simple`, `advanced`, `items` | Identico |
| `boxes[]` (incl. `kit`, `volume`) | Supportato | Identico |
| `items[]` | Supportato | Identico, **senza** `selling_unit_value` |
| `ior` (Importer of Record) | Supportato | **Non fa parte del contratto** |
| `declared_freight` | Supportato | Identico |
| `manufacturer` | Supportato | Identico |
| `level` (`simple`/`detailed`) | Supportato | Identico |
| Consultare il risultato in seguito | `GET /v1/quotation/{quotation}/{level}` | **Non esiste** |
| Warning doganali | Supportato | Identico |

### Impatto pratico sulla quotazione
- **Precisione:** in v1, città e codice postale permettono di applicare tasse e restrizioni regionali **già nel calcolo**.
  In v2, il valore può essere affinato quando l'indirizzo completo entra nella spedizione.
- **Riconsultazione:** la v2 non ha un endpoint di riconsultazione. **Conserva la risposta del `POST`** se hai bisogno
  di mostrare di nuovo la quotazione all'utente.

## Confronto della spedizione

| Tema | v1 (`POST /v1/shipping`) | v2 (`POST /v2/shipping`) |
|---|---|---|
| `quotation` | Quotazione v1 | Quotazione v2 |
| `carrier_code` / `slug` | Uno dei due | Identico |
| `address_*.street`, `name`, `email`, `phone`, `type` | Obbligatori | Identici |
| `address_*.city` | Non accettato | **Obbligatorio** (min. 3, max. 100) |
| `address_*.postal_code` | Non accettato | **Obbligatorio** (o `state`) |
| `address_*.state` / `state_code` | Non accettati | **Obbligatori** (uno dei due, o `postal_code`) |
| `payment.type` | `billed`, `gateway`, `afterward` | Identico |
| `nfkey`, `invoice_number`, `shipment_reference` | Supportati | Identici |
| `cost_center_code`, `payment_notify_shipping_billing` | Supportati | Identici |
| `attachments` | Supportato | Identico |
| `biller` | Supportato | Identico, con località validata |

## Operazioni post-creazione

Tutte le operazioni successive alla creazione della spedizione usano **gli stessi endpoint `/v1`**, indipendentemente dalla
versione in cui la spedizione è stata creata.

| Operazione | Rotta | Vale per la spedizione v1? | Vale per la spedizione v2? |
|---|---|---|---|
| Consultare lo status | `GET /v1/shipping/{shipping}/status` | Sì | **Sì** |
| Status per canale | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` | Sì | **Sì** |
| Consultare i documenti | `GET /v1/shipping/{shipping}/docs` | Sì | **Sì** |
| Documenti per canale | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` | Sì | **Sì** |
| Aggiornare (NF-e, allegati, centro di costo) | `PUT /v1/shipping/{shipping}` | Sì | **Sì** |
| Rimuovere allegati | `DELETE /v1/shipping/{shipping}/docs` | Sì | **Sì** |
| Annullare | `DELETE /v1/shipping/{shipping}` | Sì | **Sì** |
| Annullare per canale | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` | Sì | **Sì** |
| Pianificare il ritiro | `POST /v1/schedule` | Sì | **Sì** |
| Consultare la pianificazione | `GET /v1/schedule/{shipping}` | Sì | **Sì** |
| Annullare la pianificazione | `DELETE /v1/schedule/{shipping}/{reason}` | Sì | **Sì** |

> Non esistono rotte `GET`, `PUT` o `DELETE` sotto `/v2/shipping`.

## Confronto dei payload affiancati

### Quotazione

**v1 — indirizzo completo nella quotazione:**
```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
  },
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "T-shirt cotone", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

**v2 — solo paese/stato nella quotazione:**
```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "T-shirt cotone", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

### Spedizione

**v1 — senza località (è arrivata dalla quotazione):**
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
  "payment": { "type": "billed" }
}
```

**v2 — con località (non è arrivata dalla quotazione):**
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
  "payment": { "type": "billed" }
}
```

## Come scegliere

Rispondi alle domande in ordine:

1. **Al momento di quotare, conosci già la città e il codice postale di destinazione?**
   - Sì → **v1**
   - No → **v2**
2. **Hai bisogno di riconsultare la quotazione in seguito, in un'altra richiesta?**
   - Sì → **v1** (`GET /v1/quotation/{quotation}/{level}`)
   - No → una qualsiasi
3. **Usi `ior` (Importer of Record) o `selling_unit_value`?**
   - Sì → **v1**
   - No → una qualsiasi
4. **Vuoi il massimo livello di precisione delle imposte già nella quotazione?**
   - Sì → **v1**
   - No → una qualsiasi

### Scenari tipici

| Scenario | Versione consigliata | Motivo |
|---|---|---|
| Checkout di e-commerce (indirizzo già compilato) | **v1** | Quotazione precisa, mostrata al cliente finale |
| Calcolatore di nolo nella pagina del prodotto (solo paese) | **v2** | Il cliente non ha ancora indicato l'indirizzo |
| Marketplace che riceve l'ordine completo via webhook | **v1** | Tutti i dati esistono già |
| Integrazione B2B con spedizione in lotto | **v1** | Flusso operativo completo nella stessa versione |
| Simulazione in una landing page internazionale | **v2** | Payload minimo, meno frizione |
| Quotazione + spedizione + documenti in una sola chiamata | **[Manifesto](api-manifesto.md)** | Flusso sincrono, indipendente da v1/v2 |

## Altri modi per effettuare una spedizione

Oltre alla coppia quotazione + spedizione, esistono due flussi alternativi:

| Flusso | Rotta | Caratteristica |
|---|---|---|
| [Manifesto](api-manifesto.md) | `POST /v1/manifest` | **Sincrono**: quota, crea la spedizione presso il corriere e restituisce i documenti nella stessa risposta. Accetta solo il pagamento `billed` |
| [Bozza](api-rascunho.md) | `POST /v1/draft` | **Asincrono**: accetta un payload parziale, valida in seguito ed evolve verso quotazione + spedizione. Accetta il pagamento `billed` o `afterward` |

## Note finali
- Le due versioni convivono: puoi usare la v2 in vitrina e la v1 al checkout, purché **ogni quotazione
  generi la spedizione nella propria versione**.
- Gli endpoint di supporto ([Scatola](api-caixa-v1.md), [Articoli](api-itens-v1.md),
  [Contratto](api-contrato-v1.md)) sono condivisi ed esistono solo in `/v1`.
