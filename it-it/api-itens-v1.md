# Articoli

- [Torna al README della lingua](README.md)

## Panoramica
- Gestisce articoli registrati per uso in quotazioni, bozze e spedizioni.
- Gli articoli possono essere consultati, creati, aggiornati e rimossi in batch.
## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header opzionale della lingua: `lang`.

## Convenzioni
- `Content-Type: application/json`
- La consultazione usa `items[skus][]` nella query.
- Registrazione/aggiornamento/rimozione sono asincroni.

## Flusso di utilizzo
1. Elenca gli articoli già registrati.
2. Registra gli articoli base.
3. Aggiorna gli articoli quando ci sono cambiamenti di dimensione, peso o classificazione.
4. Rimuovi gli articoli obsoleti.

## Endpoints
### Elencare articoli
- Metodo e rotta: `GET /v1/item`

### Creare articoli
- Metodo e rotta: `POST /v1/item`

### Aggiornare articoli
- Metodo e rotta: `PUT /v1/item`

### Rimuovere articoli
- Metodo e rotta: `DELETE /v1/item`

## Parametri
### Elenco
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `items[skus][]` | query | array | no | Lista di SKU | `TSH-000-S` | Vuoto restituisce tutti |

### Registrazione/aggiornamento
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `items[]` | body | array | sì | Lista di articoli | - | Processamento in batch |
| `name` | body | string | sì | Nome dell'articolo | `Tenis` | Obbligatorio |
| `description` | body | string | sì | Descrizione | `Nike 44` | Fino a 155 caratteri |
| `category` | body | string | sì | Categoria | `950450` | Valore libero nella fonte |
| `country_code` | body | string | sì | Paese di origine | `BR` | ISO2 |
| `sku` | body | string | sì | SKU | `oLdTsW222` | Obbligatorio |
| `ean` | body | string | no | EAN | `7894900010015` | Opzionale |
| `height` | body | integer | sì | Altezza | `5` | - |
| `width` | body | integer | sì | Larghezza | `5` | - |
| `depth` | body | integer | sì | Profondità | `5` | - |
| `weight` | body | number | sì | Peso | `0.3` | - |
| `unit_value` | body | number | no | Valore unitario | `220.25` | Opzionale |
| `selling_unit_value` | body | number | no | Valore di vendita | `330.00` | Opzionale |
| `hscode` | body | string | sì | HS code | `950450` | Obbligatorio |
| `is_groupable` | body | boolean | sì | Può raggruppare | `true` | Obbligatorio |
| `box_code` | body | integer | no | Scatola standard | `3` | Opzionale |
| `box_identifier` | body | string | no | Scatola standard | `CX0236` | Opzionale |

### Eliminazione
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---|---|---|---|---|
| `items.skus` | body | array | sì | SKU da rimuovere | `["TSH-000-S"]` | Obbligatorio |

## Payload minimo e massimo
### Payload minimo
```json
{
  "items": [
    {
      "name": "Teste prod 1",
      "description": "Teste prod 1",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "hscode": "673057",
      "is_groupable": false
    }
  ]
}
```

### Payload massimo
```json
{
  "items": [
    {
      "name": "Teste prod 1 update",
      "description": "Teste prod 1 update",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "ean": "7894900010015",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "unit_value": 220.25,
      "selling_unit_value": 330,
      "hscode": "673057",
      "is_groupable": false,
      "box_code": 129,
      "box_identifier": "CX0236"
    }
  ]
}
```

## Esempio di request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Esempio di response
```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "name": "Tenis",
        "description": "Nike 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "oLdTsW222",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "hscode": "950450",
        "is_groupable": true
      }
    ]
  }
}
```

## Regole di business
- Gli articoli sono processati in job asincrono.
- L'elenco usa collezione per utente.
- `box_code` e `box_identifier` sono opzionali e servono come collegamento standard con scatola.

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `422` | Validazione | Correggere campi obbligatori |
| `500` | Errore imprevisto | Contattare il team responsabile |
