# Scatola

- [Torna al README della lingua](README.md)

## Panoramica
- Gestisce la registrazione e la consultazione delle scatole usate in quotazioni, bozze e spedizioni.
- Usa per standardizzare dimensioni, peso e identificazione degli imballaggi.
## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header opzionale della lingua: `lang`.

## Convenzioni
- `Content-Type: application/json`
- La consultazione usa `boxes[box_ids][]` nella query.
- Registrazione/aggiornamento avviene in modo asincrono.

## Flusso di utilizzo
1. Elenca le scatole disponibili.
2. Registra o aggiorna scatole base.
3. Rimuovi scatole quando necessario.
4. Riutilizza i codici nelle quotazioni e nelle bozze.

## Endpoints
### Elencare scatole
- Metodo e rotta: `GET /v1/box`
- Quando usare: per cercare scatole registrate o filtrare per ID.

### Creare o aggiornare scatole
- Metodo e rotta: `POST /v1/box`
- Quando usare: quando si mantiene la base di imballaggi dell'integratore.

### Rimuovere scatole
- Metodo e rotta: `DELETE /v1/box`
- Quando usare: quando si elimina una o più scatole registrate.

## Parametri
### Elenco
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `boxes[box_ids][]` | query | array | no | Lista di codici/identificatori | `1` | Vuoto restituisce tutte |

### Registrazione
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `boxes[]` | body | array | sì | Lista di scatole | - | Ogni item viene processato in batch |
| `box_code` | body | string | no | Codice della scatola | `1` | Opzionale per aggiornamento |
| `box_identifier` | body | string | sì | Identificatore della scatola | `CX0236` | Obbligatorio nella registrazione |
| `name` | body | string | sì | Nome | `Cartucho Individual` | Fino a 155 caratteri |
| `height` | body | number | sì | Altezza | `65` | >= 1 |
| `width` | body | number | sì | Larghezza | `65` | >= 1 |
| `depth` | body | number | sì | Profondità | `65` | >= 1 |
| `weight` | body | number | sì | Peso | `2.8` | Regex con fino a 3 decimali |
| `activated` | body | boolean | sì | Attivata | `true` | - |

### Eliminazione
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `boxes.box_ids` | body | array | sì | ID delle scatole | `[1, 2]` | Lista obbligatoria |

## Payload minimo e massimo
### Payload minimo
```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

### Payload massimo
```json
{
  "boxes": [
    {
      "box_code": 1,
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

## Esempio di request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "boxes": [
      {
        "box_identifier": "CX0236",
        "name": "Cartucho Individual",
        "height": 65,
        "width": 65,
        "depth": 65,
        "weight": 2.8,
        "activated": true
      }
    ]
  }'
```

## Esempio di response
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Regole di business
- La registrazione viene processata in job asincrono.
- L'elenco può essere memorizzato in cache per utente e pagina.
- L'eliminazione accetta ID multipli.

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `422` | Validazione | Correggere campi obbligatori |
| `400` | Richiesta non valida | Verificare payload | 
| `500` | Errore imprevisto | Contattare il team responsabile |
