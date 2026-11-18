# Articoli

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Mantiene la registrazione dei prodotti che spedisci, con peso, dimensioni, HS Code e valori.
- Con l'articolo registrato, la quotazione può richiamarlo solo tramite lo `sku`, senza ripetere tutti i dati.
- È ciò che abilita la quotazione `type: items` e il raggruppamento automatico `smartbox`.
- Tutte le operazioni di scrittura sono **asincrone**.

> ⚠️ **Importante:** `POST`, `PUT` e `DELETE` **non validano i campi in modo sincrono**.
> L'API risponde `200` confermando la ricezione e valida ogni articolo durante l'elaborazione.
> Gli articoli non validi vengono **scartati silenziosamente** e il motivo resta registrato nel report di
> importazione, nel pannello. Verifica sempre il risultato con `GET /v1/item`.

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

## Perché usare la registrazione degli articoli

**Senza registrazione** — ripetendo tutto in ogni quotazione:
```json
"items": [
  { "box_id": 1, "name": "T-shirt cotone S", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
]
```

**Con registrazione** — richiamando lo SKU:
```json
"items": [
  { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
]
```

Con `type: items`, non devi nemmeno indicare la scatola: la piattaforma sceglie tra le
[scatole registrate](api-caixa-v1.md).

## Flusso di utilizzo
1. Registra gli articoli con `POST /v1/item`.
2. Attendi l'elaborazione asincrona.
3. Verifica con `GET /v1/item`.
4. Usa lo `sku` nelle [quotazioni](api-cotacao-v1.md).
5. Aggiorna con `PUT /v1/item` e rimuovi con `DELETE /v1/item`.

## Endpoint: elencare gli articoli

- **Metodo e rotta:** `GET /v1/item`

| Parametro | Posizione | Tipo | Obbligatorio | Predefinito | Descrizione |
|---|---|---|---|---|---|
| `items[skus][]` | query | array | no | - | Elenco di SKU. **Vuoto o assente restituisce tutti** |
| `page` | query | integer | no | `1` | Pagina dell'elenco |

- **Elementi per pagina:** 15

```bash
# Tutti gli articoli
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'

# Filtrando per SKU
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "name": "Scarpe",
        "description": "Scarpe da corsa uomo 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "TSH-000-S",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "unit_value": 220.25,
        "selling_unit_value": 330.00,
        "hscode": "950450",
        "is_groupable": true,
        "is_foldable": false,
        "box_code": 3,
        "box_identifier": "CX0236"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

## Endpoint: creare articoli

- **Metodo e rotta:** `POST /v1/item`
- **Elaborazione:** asincrona, in lotto.

### Parametri — `items[]`

| Campo | Tipo | Obbligatorio | Min. | Max. | Descrizione |
|---|---|---|---|---|---|
| `sku` | string | **sì** | - | - | Codice univoco del prodotto nella tua base. **Non può ripetersi** |
| `name` | string | **sì** | - | - | Nome del prodotto (compare sull'invoice) |
| `description` | string | **sì** | 6 | 155 | Descrizione del prodotto. **Descrizioni distinte tra gli articoli evitano avvisi doganali** |
| `category` | string | **sì** | 6 | 6 | Categoria del prodotto. **Esattamente 6 caratteri** |
| `country_code` | string | **sì** | 2 | 2 | Paese di origine/fabbricazione in ISO 3166-1 alpha-2 |
| `hscode` | string | **sì** | - | - | NCM/HS Code. Definisce la tabella delle imposte applicata a destinazione |
| `height` | number | **sì** | 0.01 | - | Altezza del prodotto |
| `width` | number | **sì** | 0.01 | - | Larghezza del prodotto |
| `depth` | number | **sì** | 0.01 | - | Profondità del prodotto |
| `weight` | number | **sì** | 0.01 | - | Peso unitario |
| `is_groupable` | boolean | **sì** | - | - | `true` permette di raggruppare l'articolo con altri nella stessa scatola. `false` forza una scatola esclusiva nello `smartbox` |
| `ean` | string | no | - | - | Codice EAN/GTIN |
| `unit_value` | number | no | - | - | Valore unitario predefinito. Se assente nella registrazione **e** nella quotazione, la quotazione fallisce |
| `selling_unit_value` | number | no | - | - | Valore di vendita unitario. Necessario quando la quotazione usa `ior: true` |
| `box_code` | integer | no | - | - | Scatola predefinita dell'articolo ([Scatola](api-caixa-v1.md)). Deve essere un **intero** |
| `box_identifier` | string | no | - | - | Scatola predefinita dell'articolo, tramite l'identificatore testuale |
| `is_foldable` | boolean | no | - | - | Articolo pieghevole. Nello `smartbox`, i pieghevoli vengono sistemati per volume; i non pieghevoli per dimensioni |

> **Dimensioni e peso** seguono il sistema di misura dell'account/della quotazione: cm/kg con `metric`,
> in/lb con `imperial`.

### Payload minimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "T-shirt cotone S",
      "description": "T-shirt 100% cotone manica corta bianca taglia S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "is_groupable": true
    }
  ]
}
```

### Payload massimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "T-shirt cotone S",
      "description": "T-shirt 100% cotone manica corta bianca taglia S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3,
      "box_identifier": "CX0236"
    },
    {
      "sku": "SNK-044",
      "name": "Scarpe sportive 44",
      "description": "Scarpe da corsa uomo numero 44 colore nero",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 199.9,
      "selling_unit_value": 349.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

### Esempio di request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "T-shirt cotone S", "description": "T-shirt 100% cotone manica corta bianca taglia S", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint: aggiornare articoli

- **Metodo e rotta:** `PUT /v1/item`
- **Chiave di aggiornamento:** lo `sku`. Invia lo SKU esistente e i campi da modificare.
- **Elaborazione:** asincrona, in lotto.
- **Struttura del payload:** identica a quella del `POST`.

> ⚠️ **Uno SKU inesistente non viene creato.** L'aggiornamento non fa "upsert": se lo SKU non esiste,
> non accade nulla (e nulla viene segnalato nella risposta HTTP). Usa `POST` per creare.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "T-shirt cotone S", "description": "T-shirt premium 100% cotone manica corta bianca S", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint: rimuovere articoli

- **Metodo e rotta:** `DELETE /v1/item`
- **Elaborazione:** asincrona, in lotto.

| Campo | Tipo | Obbligatorio | Descrizione |
|---|---|---|---|
| `items` | object | **sì** | Involucro |
| `items.skus` | array | **sì** | Elenco degli SKU da rimuovere |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Validazione asincrona — tabella di riferimento

Poiché la validazione avviene durante l'elaborazione, usa questa tabella per assicurarti che gli articoli vengano accettati.
La prima regola violata scarta l'articolo.

| Campo | Regola | Motivo registrato nel report |
|---|---|---|
| (articolo) | Deve essere un oggetto | `not_array` |
| `name` | Obbligatorio, testo | `name` |
| `description` | Obbligatoria, testo | `description` |
| `category` | Obbligatoria, testo, **esattamente 6 caratteri** | `category` |
| `country_code` | Obbligatorio, testo, **esattamente 2 caratteri** | `country_code` |
| `sku` | Obbligatorio, testo | `sku` |
| `ean` | Se inviato, deve essere testo | `ean` |
| `height` | Obbligatoria, numerica, **≥ 0.01** | `height` |
| `width` | Obbligatoria, numerica, **≥ 0.01** | `width` |
| `depth` | Obbligatoria, numerica, **≥ 0.01** | `depth` |
| `weight` | Obbligatorio, numerico, **≥ 0.01** | `weight` |
| `hscode` | Obbligatorio, testo | `hscode` |
| `is_groupable` | Se inviato, deve essere booleano | `is_groupable` |
| `box_code` | Se inviato, deve essere un **intero** (non una stringa) | `box_code` |
| `box_identifier` | Se inviato, deve essere testo | `box_identifier` |
| `unit_value` | Se inviato, deve essere numerico | `unit_value` |
| `selling_unit_value` | Se inviato, deve essere numerico | `selling_unit_value` |

Altre situazioni registrate nel report:

| Situazione | Comportamento |
|---|---|
| `items` vuoto o non è un array | Report: `Itens isn't an array or is empty` |
| SKU duplicato nella registrazione | Il record viene rifiutato (lo SKU è un indice univoco) |
| SKU inesistente nel `PUT` | Nulla viene modificato e nulla viene creato |
| SKU non valido nel `DELETE` | Report: `sku_invalid` |

## Come usare gli articoli nella quotazione

| `type` della quotazione | Payload in `items[]` | Comportamento |
|---|---|---|
| `advanced` | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | Definisci tu in quale scatola va l'articolo |
| `items` | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | La piattaforma sceglie le scatole (raggruppamento dell'account) |
| qualsiasi | `{ "box_id": 1, "name": "...", "country_code": "BR", "hscode": "...", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | Articolo occasionale, senza registrazione |

Vedi la sezione `items[]` in [Quotazione (v1)](api-cotacao-v1.md#parametri--items).

## Regole di business
- Tutte le operazioni di scrittura girano in una **coda asincrona**.
- Lo `sku` è un **indice univoco** per account.
- Gli articoli sono **per account**: ogni utente vede solo i propri.
- La registrazione deve essere completa perché la quotazione funzioni. Messaggi correlati:
  - `Nenhum item foi encontrado com o SKU :sku especificado !`
  - `Por favor verifique o cadastro do item :sku, pois poder haver informações imcompletas!`
  - `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).`
  - `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...`
- `is_groupable: false` garantisce una scatola esclusiva nel raggruppamento `smartbox`.
- Uno `hscode` errato genera un errore di imposta nella quotazione:
  `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item).`

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `200` con articoli assenti | Articolo scartato nella validazione asincrona | Verificare il report nel pannello e la tabella di validazione sopra |
| `403` | Permesso mancante per l'azione | Richiedere il permesso al supporto |
| `422` | `items` assente | Inviare l'array `items` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- Poiché la validazione non è sincrona, **verifica sempre con `GET /v1/item`** dopo aver registrato o aggiornato.
- Mantieni descrizioni distinte tra prodotti diversi: le descrizioni ripetute generano il warning
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` nella quotazione.
- Per standardizzare gli imballaggi usati con questi articoli, vedi [Scatola](api-caixa-v1.md).
