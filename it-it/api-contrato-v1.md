# Contratto

- [Torna al README della lingua](README.md)

## Panoramica
- Elenca i contratti disponibili per l'utente autenticato.
- I contratti sono usati nella quotazione e nella creazione di spedizioni.
## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header opzionale della lingua: `lang`.

## Convenzioni
- `Content-Type: application/json`
- L'elenco può filtrare per `status`.

## Flusso di utilizzo
1. Autentica l'utente.
2. Elenca i contratti disponibili.
3. Usa il `code` del contratto nella quotazione o spedizione.

## Endpoints
### Elencare contratti
- Metodo e rotta: `GET /v1/contract`
- Quando usare: per identificare contratti attivi/inattivi prima della quotazione.

## Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `status` | query | string | no | Filtra contratti | `active` | `active`, `inactive`, `all` |

## Payload minimo e massimo
- Non si applica all'endpoint di elenco.

## Esempio di request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=active' \
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
        "code": 3,
        "contract_description": "Contrato padrão PRO",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "Carrier Name",
        "initial_date": "2023-03-06 00:00:00",
        "final_date": "2023-06-30 23:59:59",
        "active": "active",
        "direct_quotation": false
      }
    ]
  }
}
```

## Regole di business
- Se non ci sono contratti nel primo filtro, l'API tenta una seconda ricerca con comportamento alternativo del backend.
- Il ritorno dipende dal collegamento dell'utente con i contratti.

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `500` | Errore imprevisto | Contattare il team responsabile |
